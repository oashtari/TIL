[2/8/23](#february-8-2023)<br>
[2/9/23](#february-9-2023)<br>
[2/10/23](#february-10-2023)<br>
[2/11/23](#february-11-2023)<br>
[2/13/23](#february-13-2023)<br>
[2/14/23](#february-14-2023)<br>
[2/18/23](#february-18-2023)<br>
[2/19/23](#february-19-2023)<br>
[2/20/23](#february-20-2023)<br>
[2/21/23](#february-21-2023)<br>
[2/22/23](#february-22-2023)<br>
[2/24/23](#february-24-2023)<br>
[2/28/23](#february-28-2023)<br>

[week of 2/27/23](/TIL/Rust/week_of_2_27_23.md)
/Users/omid/TIL/TIL/Rust/week_of_2_27_23.md
[previous week](/React/template_files.md)
/Users/omid/TIL/TIL/React/template_files.md
# February 8, 2023 

## Setup

### Create new project
   
    cargo new <project name> 

### Speed up linking phase 

- On MacOS

        brew install michaeleisel/zld/zld

        [target.x86_64-apple-darwin]
        rustflags = ["-C", "link-arg=-fuse-ld=/usr/local/bin/zld"]
        [target.aarch64-apple-darwin]
        rustflags = ["-C", "link-arg=-fuse-ld=/usr/local/bin/zld"]

### Reduce perceived compilation time

        cargo install cargo-watch

- to run after each code change

        cargo watch -x check

- to run check, then test, then run

        cargo watch -x check -x test -x run

### Checking test coverage in your code

At the time of writing tarpaulin only supports
x86_64 CPU architectures running Linux.
        
        cargo install cargo-tarpaulin

- to check coverage

        cargo tarpaulin --ignore-tests

## CI

### Linting

        rustup component add clippy

- run without getting warnings

        cargo clippy -- -D warnings

### Formatting

        rustup component add rustfmt

- format whole project with 

        cargo fmt

- in our CI pipeline, we'll simply add a formatting step

        cargo fmt -- --check

### Security vulnerabilities

        cargo install cargo-audit

- once installed, run

        cargo audit

## Web framework
[actix-web](https://actix.rs/)

[actix-web docs](https://docs.rs/actix-web/4.0.1/actix_web/index.html)

[axtic-web examples](https://github.com/actix/examples)

        #[tokio::main]
        async fn main() -> std::io::Result<()> {
            HttpServer::new(|| { App::new()
            .route("/", web::get().to(greet)) .route("/{name}", web::get().to(greet))
                })
                .bind("127.0.0.1:8000")?
                .run()
                .await
            }

[App](https://docs.rs/actix-web/4.0.1/actix_web/struct.App.html) is where all your application logic lives: routing, middlewares, request handlers, etc.
App is the component whose job is to take an incoming request as input and spit out a response.

We need main to be asynchronous because HttpServer::run is an asynchronous method but main, the entry- point of our binary, cannot be an asynchronous function. 

tokio::main is a procedural macro and this is a great opportunity to introduce cargo expand, an awesome addition to our Swiss army knife for Rust development:

        cargo install cargo-expand

Rust macros operate at the token level: they take in a stream of symbols (e.g. in our case, the whole main function) and output a stream of new symbols which then gets passed to the compiler. In other words, the main purpose of Rust macros is code generation.

That is exactly where cargo expand shines: it expands all macros in your code without passing the output to the compiler, allowing you to step through it and understand what is going on. You can run it with:

        cargo expand

Cargo expand relies on the nightly compiler to expand macros, install the nightly compiler with:

        rustup toolchain install nightly --allow-downgrade

Some components of the bundle installed by rustup might be broken/missing on the latest nightly release: --allow-downgrade tells rustup to find and install the latest nightly where all the needed components are available.

Use the nightly toolchain just for this command invocation
        cargo +nightly expand

In other words, the job of #[tokio::main] is to give us the illusion of being able to define an asynchronous main while, under the hood, it just takes our main asynchronous body and writes the necessary boilerplate to make it run on top of tokio’s runtime.

# February 9 2023 

:(

# February 10 2023 

Can do route specific manual tests with cli commands. Launch the application first in another terminal with `cargo run`
        curl -v http://127.0.0.1:8000/health_check

### Integration testing

Testing the API by interacting with it in the same exact way a user would: performing HTTP requests against it and verifying our assumptions on the responses we receive.

This is often referred to as black box testing: we verify the behaviour of a system by examining its output given a set of inputs without having access to the details of its internal implementation.

-- 

We will opt for a fully black-box solution: we will launch our application at the beginning of each test and interact with it using an off-the-shelf HTTP client (e.g. reqwest).

-- 

Test locations:

#### embedded test modules -- is part of your project, just hidden behind a configuration conditional check, #[cfg(test)].

// Some code I want to test
        #[cfg(test)]
        mod tests {
            // Import the code I want to test
        use super::*; // My tests
        }

#### external tests folder 
        src/
        tests/
        Cargo.toml
        Cargo.lock

#### part of public documentation

        /// Check if a number is even.
        /// ```rust
        /// use zero2prod::is_even;
        ///
        /// assert!(is_even(2));
        /// assert!(!is_even(1));
        /// ```
        pub fn is_even(x: u64) -> bool {
        x % 2 == 0 }

Anything under the tests folder and your documentation tests, instead, are compiled in their own separate binaries.
This has consequences when it comes to visibility rules.

We need to refactor our project into a library and a binary: all our logic will live in the library crate while the binary itself will be just an entrypoint with a very slim main function.

        [package]
        name = "zero2prod"
        version = "0.1.0"
        authors = ["Luca Palmieri <contact@lpalmieri.com>"]
        edition = "2021"
        [dependencies]
        # [...]

We are relying on cargo’s default behaviour: unless something is spelled out, it will look for a src/main.rs file as the binary entrypoint and use the package.name field as the binary name.
Looking at the manifest target specification, we need to add a lib section to add a library to our project:

        [package]
        name = "zero2prod"
        version = "0.1.0"
        authors = ["Luca Palmieri <contact@lpalmieri.com>"]
        edition = "2021"
        [lib]
        # We could use any path here, but we are following the community convention
        # We could specify a library name using the `name` field. If unspecified,
        # cargo will default to `package.name`, which is what we want.
        path = "src/lib.rs"
        [dependencies]
        # [...]

Notice the double square brackets: it's an array in TOML's syntax.
We can only have one library in a project, but we can have multiple binaries!
If you want to manage multiple libraries in the same repository have a look at the workspace feature - we'll cover it later on.
        [[bin]]
        path = "src/main.rs"
        name = "zero2prod"

`tokio::test` is the testing equivalent of `tokio::main`.
It also spares you from having to specify the `#[test]` attribute.

You can inspect what code gets generated using
`cargo expand --test health_check` (<- name of the test file)

Sample test:

        #[tokio::test]
        async fn health_check_works() {
            // Arrange
            spawn_app().await.expect("Failed to spawn our app."); 
            // We need to bring in `reqwest` to perform HTTP requests against our application. let client = reqwest::Client::new();

            // Act
                let response = client
                    .get("http://127.0.0.1:8000/health_check")
                    .send()
                    .await
                    .expect("Failed to execute request.");

            // Assert
            assert!(response.status().is_success());
            assert_eq!(Some(0), response.content_length()); }
            
            // Launch our application in the background ~somehow~
            async fn spawn_app() -> std::io::Result<()> { todo!()
        }

Dev dependencies are used exclusively when running tests or examples
They do not get included in the final application binary!
[dev-dependencies]
reqwest = "0.11"

spawn_app is the only piece that will, reasonably, depend on our application code.

        async fn spawn_app() -> std::io::Result<()> { 
            zero2prod::run().await
        }

We need to run our application as a background task.
tokio::spawn comes quite handy here: tokio::spawn takes a future and hands it over to the runtime for polling, 
without waiting for its completion; 
it therefore runs concurrently with downstream futures and tasks (e.g. our test logic).

// Notice the different signature!
// We return `Server` on the happy path and we dropped the `async` keyword // We have no .await call, so it is not needed anymore.
        pub fn run() -> Result<Server, std::io::Error> {
            let server = HttpServer::new(|| { 
                App::new()
                .route("/health_check", web::get().to(health_check))
                    })
                .bind("127.0.0.1:8000")?
                .run();
                // No .await here!
            Ok(server)
        }

No .await call, therefore no need for `spawn_app` to be async now.
We are also running tests, so it is not worth it to propagate errors:
if we fail to perform the required setup we can just panic and crash all the things.
        fn spawn_app() {
                let server = zero2prod::run()
                    .expect("Failed to bind address"); 
                    // Launch the server as a background task
                    // tokio::spawn returns a handle to the spawned future,
                    // but we have no use for it here, hence the non-binding let let _ = tokio::spawn(server);
            }

tokio::test spins up a new runtime at the beginning of each test case and they shut down at the end of each test case.

We need our port to be dynamic, not hard-coded.
The operating system comes to the rescue: we will be using port 0.
Port 0 is special-cased at the OS level: trying to bind port 0 will trigger an OS scan for an available port which will then be bound to the application.

We need, somehow, to find out what port the OS has gifted our application and return it from spawn_app.
There are a few ways to go about it - we will use a std::net::TcpListener.
Our HttpServer right now is doing double duty: given an address, it will bind it and then start the applic- ation. We can take over the first step: we will bind the port on our own with TcpListener and then hand that over to the HttpServer using listen.

# February 11, 2023 

Figured out my .gitconfigure was not set up, so my work was not getting me green dots on GitHub :(

# February 12, 2023

Day off.

# February 13, 2023 

### Working with HTML Forms

There are a few options available when using HTML forms: application/x-www-form-urlencoded is the most suitable to our usecase.
Quoting MDN web docs, with application/x-www-form-urlencoded
        "the keys and values [in our form] are encoded in key-value tuples separated by ‘&’, with a ‘=’ between
        the key and the value. Non-alphanumeric characters in both keys and values are percent encoded."

#### Parsing form data from a POST request

A 404 means it's not hitting an endpoint, so we need to add a .route for the path we want.

        pub fn run(listener: TcpListener) -> Result<Server, std::io::Error> { let server = HttpServer::new(|| {
            App::new()
            .route("/health_check", web::get().to(health_check))
            // A new entry in our routing table for POST /subscriptions requests .route("/subscriptions", web::post().to(subscribe))
            }) .listen(listener)? .run();
        
            Ok(server)
        }

We need to implement Extractors. Extractors are used, as the name implies, to tell the framework to extract certain pieces of information from an incoming request.
actix-web provides several extractors out of the box to cater for the most common usecases:
    • Path to get dynamic path segments from a request’s path; 
    • Query for query parameters;
    • Json to parse a JSON-encoded request body;
    • etc.

    Form data helper (application/x-www-form-urlencoded).
    Can be used to extract url-encoded data from the request body, or send url-encoded data as the re- sponse.

    An extractor can be accessed as an argument to a handler function. Actix-web supports up to 10
    extractors per handler function. Argument position does not matter.

#### Serialization

    "Serde is a framework for serializing and deserializing Rust data structures efficiently and generically."

### Database setup

To run queries in our test suite we need:
    • a running Postgres instance12;
    • atabletostoreoursubscribersdata.

Install Postgres, which should include PG Admin 4.
Run in terminal:
        export PATH="/Library/PostgreSQL/15/bin:$PATH"

Install sqlx
        cargo install --version="~0.6" sqlx-cli --no-default-features --features rustls,postgres

Launch postgres by running:

        ./scripts/init_db.sh

Create a database:

        sqlx-database-create

# February 14, 2023 

To migrate database tables:

        sqlx migrate run

To migrate by skipping Docker:

        SKIP_DOCKER=true ./scripts/init_db.sh

Add sqlx as a dependency to cargo.toml

        [dependencies.sqlx] version = "0.6" default-features = false features = [
            "runtime-tokio-rustls",
            "macros",
            "postgres",
            "uuid",
            "chrono",
            "migrate"
        ]

Yeah, there are a lot of feature flags. Let’s go through all of them one by one:
    • runtime-tokio-rustls tells sqlx to use the tokio runtime for its futures and rustls as TLS backend;
    • macros gives us access to sqlx::query! and sqlx::query_as!, which we will be using extensively;
    • postgres unlocks Postgres-specific functionality (e.g. non-standard SQL types);
    • uuid adds support for mapping SQL UUIDs to the Uuid type from the uuid crate. We need it to work
    with our id column;
    • chrono adds support for mapping SQL timestamptz to the DateTime<T> type from the chrono crate.
    We need it to work with our subscribed_at column;
    • migrate gives us access to the same functions used under the hood by sqlx-cli to manage migrations.
    It will turn out to be useful for our test suite.

#### Configuration Management for a DB

The simplest entrypoint to connect to a Postgres database is PgConnection.

PgConnection implements the Connection trait which provides us with a connect method: it takes as input a connection string and returns us, asynchronously, a Result<PostgresConnection, sqlx::Error>.

The config crate is Rust’s swiss-army knife when it comes to configuration: it supports multiple file formats and it lets you combine different sources hierarchically (e.g. environment variables, configuration files, etc.) to easily customise the behaviour of your application for each deployment environment.

### File structure

src/
    configuration.rs
    lib.rs
    main.rs
    routes/
        mod.rs
        health_check.rs
        subscriptions.rs
    startup.rs

startup.rs will host our run function
health_check goes into routes/health_check.rs
subscribe and FormData into routes/subscriptions.rs
configuration.rs starts empty. 

#### Reading a configuration file

Tomanageconfigurationwithconfigwemustrepresentour application settings as a Rust type that implements serde’s Deserialize trait.

Let’s create a new Settings struct:

        #[derive(serde::Deserialize)] 
        pub struct Settings {}

We have two groups of configuration values at the moment:
    • the application port, where actix-web is listening for incoming requests (currently hard-coded to 8000 in main.rs);
    • the database connection parameters.

Make sure to add config as a dependency.

We want to read our application settings from a configuration file named configuration.yaml.

# February 18, 2023 

#### Connecting to Postgres

PgConnection::connect wants a single connection string as input, while DatabaseSettings provides us with granular access to all the connection parameters. Let’s add a convenient connection_string method to do it:

                //! src/configuration.rs
                // [...]
                impl DatabaseSettings {
                pub fn connection_string(&self) -> String { format!(
                        "postgres://{}:{}@{}:{}/{}",
                        self.username, self.password, self.host, self.port, self.database_name
                        )
                } }

As we discussed before, sqlx reaches out to Postgres at compile-time to check that queries are well-formed. Just like sqlx-cli commands, it relies on the DATABASE_URL environment variable to know where to find the database.
We could export DATABASE_URL manually, but we would then run in the same issue every time we boot our machine and start working on this project. Let’s take the advice of sqlx’s authors - we’ll add a top-level .env file

                DATABASE_URL="postgres://postgres:password@localhost:5432/newsletter"

sqlx will read DATABASE_URL from it and save us the hassle of re-exporting the environment variable every single time.

It feels a bit dirty to have the database connection parameters in two places (.env and configuration.yaml), but it is not a major problem: configuration.yaml can be used to alter the runtime behaviour of the ap- plication after it has been compiled, while .env is only relevant for our development process, build and test steps.
Commit the .env file to version control - we will need it in CI soon enough!

If you check on it, you will notice that your CI pipeline is now failing to perform most of the checks we introduced at the beginning of our journey.
Our tests now rely on a running Postgres database to be executed properly. All our build commands (cargo check, cargo lint, cargo build), due to sqlx’s compile-time checks, need an up-and-running database!
We do not want to venture further with a broken CI.
You can find an updated version of the GitHub Actions setup on GitHub. Only general.yml needs to be updated.

#### Persisting a new subscriber

To execute a query within subscribe we need to get our hands on a database connection.

#### application state in actix-web

actix-web gives us the possibility to attach to the application other pieces of data that are not related to the lifecycle of a single incoming request - the so-called application state.

You can add information to the application state using the app_data method on App.

Let’s try to use app_data to register a PgConnection as part of our application state. We need to modify our
run method to accept a PgConnection alongside the TcpListener:

#### atcix-web Workers

HttpServer::new does not take App as argument - it wants a closure that returns an App struct.

This is to support actix-web’s runtime model: actix-web will spin up a worker process for each available core on your machine.

Each worker runs its own copy of the application built by HttpServer calling the very same closure that HttpServer::new takes as argument.

That is why connection has to be cloneable - we need to have one for every copy of App.
But, as we said, PgConnection does not implement Clone because it sits on top of a non-cloneable system resource, a TCP connection with Postgres. What do we do?

We can use web::Data, another actix-web extractor.
web::Data wraps our connection in an Atomic Reference Counted pointer, an Arc: each instance of the application, instead of getting a raw copy of a PgConnection, will get a pointer to one.

Arc<T> is always cloneable, no matter who T is: cloning an Arc increments the number of active references and hands over a new copy of the memory address of the wrapped value.

Handlers can then access the application state using the same extractor.

#### data extractor

We called Data an extractor, but what is it extracting a PgConnection from?

actix-web uses a type-map to represent its application state: a HashMap that stores arbitrary data (using the Any type) against their unique type identifier (obtained via TypeId::of).

web::Data, when a new request comes in, computes the TypeId of the type you specified in the signature (in our case PgConnection) and checks if there is a record corresponding to it in the type-map. If there is one, it casts the retrieved Any value to the type you specified (TypeId is unique, nothing to worry about) and passes it to your handler.

It is an interesting technique to perform what in other language ecosystems might be referred to as dependency injection.

#### The Insert query

Let’s unpack what is happening:
        • we are binding dynamic data to our INSERT query. $1 refers to the first argument passed to query! after the query itself, $2 to the second and so forth. query! verifies at compile-time that the provided number of arguments matches what the query expects as well as that their types are compatible (e.g. you can’t pass a number as id);
        • wearegeneratingarandomUuidforid;
        • weareusingthecurrenttimestampintheUtctimezoneforsubscribed_at.

We have to add two new dependencies as well to our Cargo.toml to fix the obvious compiler errors:

                uuid = { version = "1", features = ["v4"] }
                chrono = { version = "0.4.22", default-features = false, features = ["clock"] }

# February 19, 2023 

sqlx has an asynchronous interface, but it does not allow you to run multiple queries concurrently over the same database connection.

Requiring a mutable reference allows them to enforce this guarantee in their API. You can think of a mutable reference as a unique reference: the compiler guarantees to execute that they have indeed exclusive access to that PgConnection because there cannot be two active mutable references to the same value at the same time in the whole program. Quite neat.

Let’s take a second look at the documentation for sqlx’s Executor trait: what else implements Executor apart from &mut PgConnection?
Bingo: a shared reference to PgPool.

                // The function is asynchronous now!
                async fn spawn_app() -> TestApp {
                        let listener = TcpListener::bind("127.0.0.1:0").expect("Failed to bind random port");

                        let port = listener.local_addr().unwrap().port(); let address = format!("http://127.0.0.1:{}", port);
                        
                        let configuration = get_configuration().expect("Failed to read configuration.");
                        let connection_pool = PgPool::connect(
                                &configuration.database.connection_string()
                                )
                                .await
                                .expect("Failed to connect to Postgres.");
                        
                        let server = run(listener, connection_pool.clone())
                                .expect("Failed to bind address");
                        
                        let _ = tokio::spawn(server); TestApp {
                                address,
                                db_pool: connection_pool,
                        }
                }

#### Test Isolation

Your database is a gigantic global variable: all your tests are interacting with it and whatever they leave behind will be available to other tests in the suite as well as to the following test runs.

You really do not want to have any kind of interaction between your tests: it makes your test runs non- deterministic and it leads down the line to spurious test failures that are extremely tricky to hunt down and fix.
There are two techniques I am aware of to ensure test isolation when interacting with a relational database in a test:
        • wrap the whole test in a SQL transaction and roll back at the end of it; 
        • spin up a brand-new logical database for each integration test.

Before each test run, we want to:
        • create a new logical database with a unique name;
        • run database migrations on it.

The best place to do this is spawn_app, before launching our actix-web test application.

configuration.database.connection_string() uses the database_name specified in our configura- tion.yaml file - the same for all tests.
Let’s randomise it with:
                let mut configuration = get_configuration().expect("Failed to read configuration."); configuration.database.database_name = Uuid::new_v4().to_string();
                let connection_pool = PgPool::connect( &configuration.database.connection_string()
                )
                .await
                .expect("Failed to connect to Postgres.");

cargo test will fail: there is no database ready to accept connections using the name we generated. Let’s add a connection_string_without_db method to our DatabaseSettings:

Omitting the database name we connect to the Postgres instance, not a specific logical database. We can now use that connection to create the database we need and run migrations on it:

sqlx::migrate! is the same macro used by sqlx-cli when executing sqlx migrate run - no need to throw bash scripts into the mix to achieve the same result.

You might have noticed that we do not perform any clean-up step at the end of our tests - the logical databases we create are not being deleted. This is intentional: we could add a clean-up step, but our Postgres instance is used only for test purposes and it’s easy enough to restart it if, after hundreds of test runs, performance starts to suffer due to the number of lingering (almost empty) databases.


# February 20, 2023 

### Telemetry

#### Logs and the Log Crate

Log crate's five macros:  trace, debug, info, warn and error.

actix_web provides a Logger middleware. It emits a log record for every incoming request.

                use actix_web::middleware::Logger;

command for checking PORTs 

                sudo lsof -iTCP -sTCP:LISTEN -n -P

Add dependency:

                env_logger = "0.9"

env_logger::Logger prints log records to the terminal, using the following format:

                [<timestamp> <level> <module path>] <log message>

RUST_LOG=debug cargo run, for example, will surface all logs at debug-level or higher emitted by our applic- ation or the crates we are using.

RUST_LOG=zero2prod, instead, would filter out all records emitted by our dependencies

#### Interactions with external systems

Let’s start with a tried-and-tested rule of thumb: any interaction with external systems over the network should be closely monitored. We might experience networking issues, the database might be unavailable, queries might get slower over time as the subscribers table gets longer, etc.

Logs need to be correlated

We need a way to correlate all logs related to the same request.
This is usually achieved using a request id (also known as correlation id): when we start to process an in- coming request we generate a random identifier (e.g. a UUID) which is then associated to all logs concerning the fulfilling of that specific request.

#### Structred logging

The tracing crate.

tracing expands upon logging-style diagnostics by allowing libraries and applications to record structured events with additional information about temporality and causality — unlike a log mes- sage, a span in tracing has a beginning and end time, may be entered and exited by the flow of execu- tion, and may exist within a nested tree of similar spans.

##### tracing's span

                pub async fn subscribe(/* */) -> HttpResponse {
                        let request_id = Uuid::new_v4();
                        // Spans, like logs, have an associated level // `info_span` creates a span at the info-level let request_span = tracing::info_span!(
                                "Adding a new subscriber.",
                                %request_id,
                                subscriber_email = %form.email,
                                subscriber_name = %form.name
                        );
                        // Using `enter` in an async function is a recipe for disaster!
                        // Bear with me for now, but don't do this at home.
                        // See the following section on `Instrumenting Futures`
                        let _request_span_guard = request_span.enter();
                        // [...]
                        // `_request_span_guard` is dropped at the end of `subscribe`
                        // That's when we "exit" the span
                        }

        We are using the info_span! macro to create a new span and attach some values to its context: request_id, form.email and form.name.

        We are not using string interpolation anymore: tracing allows us to associate structured information to our spans as a collection of key-value pairs2. We can explicitly name them (e.g. subscriber_email for form.email) or implicitly use the variable name as key (e.g. the isolated request_id is equivalent to re- quest_id = request_id).

        Notice that we prefixed all of them with a % symbol: we are telling tracing to use their Display implement- ation for logging purposes. You can find more details on the other available options in their documentation.

        info_span returns the newly created span, but we have to explicit step into it using the .enter() method to
        activate it.

        .enter() returns an instance of Entered, a guard: as long the guard variable is not dropped all downstream spans and log events will be registered as children of the entered span. This is a typical Rust pattern, often referred to as Resource Acquisition Is Initialization (RAII): the compiler keeps track of the lifetime of all variables and when they go out of scope it inserts a call to their destructor, Drop::drop.

        The default implementation of the Drop trait simply takes care of releasing the resources owned by that variable. We can, though, specify a custom Drop implementation to perform other cleanup operations on drop - e.g. exiting from a span when the Entered guard gets dropped:

        Inspecting the source code of your dependencies can often expose some gold nuggets - we just found out that if the log feature flag is enabled tracing will emit a trace-level log when a span exits.

        run     RUST_LOG=trace cargo run


        You can enter (and exit) a span multiple times. Closing, instead, is final: it happens when the span itself is dropped.
        This comes pretty handy when you have a unit of work that can be paused and then resumed - e.g. an asyn- chronous task!

#### Instrumenting features

The executor might have to poll its future more than once to drive it to completion - while that future is idle, we are going to make progress on other futures.

This can clearly cause issues: how do we make sure we don’t mix their respective spans?

The best way would be to closely mimic the future’s lifecycle: we should enter into the span associated to our future every time it is polled by the executor and exit every time it gets parked.

That’s where Instrument comes into the picture. It is an extension trait for futures. Instrument::instrument does exactly what we want: enters the span we pass as argument every time self, the future, is polled; it exits the span every time the future is parked.

#### Tracing's subscriber

env_logger’s logger implements log’s Log trait - it knows nothing about the rich structure exposed by tra- cing’s Span!
tracing’s compatibility with log was great to get off the ground, but it is now time to replace env_logger with a tracing-native solution.

The tracing crate follows the same facade pattern used by log - you can freely use its macros to instrument your code, but applications are in charge to spell out how that span telemetry data should be processed. 

Subscriber is the tracing counterpart of log’s Log: an implementation of the Subscriber trait exposes a variety of methods to manage every stage of the lifecycle of a Span - creation, enter/exit, closure, etc.

tracing does not provide any subscriber out of the box.
We need to look into tracing-subscriber, another crate maintained in-tree by the tracing project, to find a few basic subscribers to get off the ground.

tracing-subscriber does much more than providing us with a few handy subscribers. It introduces another key trait into the picture, Layer.

Layer makes it possible to build a processing pipeline for spans data: we are not forced to provide an all- encompassing subscriber that does everything we want; we can instead combine multiple smaller layers to obtain the processing pipeline we need.

This substantially reduces duplication across in tracing ecosystem: people are focused on adding new capab- ilities by churning out new layers rather than trying to build the best-possible-batteries-included subscriber.

The cornerstone of the layering approach is Registry.
Registry implements the Subscriber trait and takes care of all the difficult stuff:

        Registry does not actually record traces itself: instead, it collects and stores span data that is exposed to any layer wrapping it [...]. The Registry is responsible for storing span metadata, recording rela- tionships between spans, and tracking which spans are active and which are closed.

        • tracing_subscriber::filter::EnvFilter discards spans based on their log levels and their origins, just as we did in env_logger via the RUST_LOG environment variable;
        • tracing_bunyan_formatter::JsonStorageLayer processes spans data and stores the associated metadata in an easy-to-consume JSON format for downstream layers. It does, in particular, propagate context from parent spans to their children;
        • tracing_bunyan_formatter::BunyanFormatterLayer builds on top of JsonStorageLayer and out- puts log records in bunyan-compatible JSON format.

tracing-bunyan-formatter also provides duration out-of-the-box: every time a span is closed a JSON mes- sage is printed to the console with an elapsed_millisecond property attached to it.

#### tracing-log

log does not emit tracing events out of the box and does not provide a feature flag to enable this behaviour.
If we want it, we need to explicitly register a logger implementation to redirect logs to our tracing subscriber for processing.
We can use LogTracer, provided by the tracing-log crate.

#### Removing unused dependencies

                cargo install cargo-udeps

cargo-udeps scans your Cargo.toml file and checks if all the crates listed under [dependencies] have actu- ally been used in the project. Check cargo-deps’ trophy case for a long list of popular Rust projects where cargo-udeps was able to spot unused dependencies and cut down build times.

                # cargo-udeps requires the nightly compiler.
                # We add +nightly to our cargo invocation
                # to tell cargo explicitly what toolchain we want to use.
                cargo +nightly udeps

#### Logs for integration testing

As a rule of thumb, everything we use in our application should be reflected in our integration tests.

init_subscriber should only be called once, but it is being invoked by all our tests. We can use once_cell to rectify it5

cargo test solves the very same problem for println/print statements. By default, it swallows everything that is printed to console. You can explicitly opt in to look at those print statements using cargo test -- --nocapture.

We need an equivalent strategy for our tracing instrumentation.
Let’s add a new parameter to get_subscriber to allow customisation of what sink logs should be written to:

In our test suite we will choose the sink dynamically according to an environment variable, TEST_LOG. If TEST_LOG is set, we use std::io::stdout.
If TEST_LOG is not set, we send all logs into the void using std::io::sink.
Our own home-made version of the --nocapture flag.

When you want to see all logs coming out of a certain test case to debug it you can run

                # We are using the `bunyan` CLI to prettify the outputted logs
                # The original `bunyan` requires NPM, but you can install a Rust-port with
                # `cargo install bunyan`
                TEST_LOG=true cargo test health_check_works | bunyan

In other words, we’d like to wrap the subscribe function in a span.
This requirement is fairly common: extracting each sub-task in its own function is a common way to struc- ture routines to improve readability and make it easier to write tests; therefore we will often want to attach a span to a function declaration.
tracing caters for this specific usecase with its tracing::instrument procedural macro. Let’s see it in ac- tion:

#[tracing::instrument] creates a span at the beginning of the function invocation and automatically at- taches all arguments passed to the function to the context of the span - in our case, form and pool. Often function arguments won’t be displayable on log records (e.g. pool) or we’d like to specify more explicitly what should/how they should be captured (e.g. naming each field of form) - we can explicitly tell tracing to ignore them using the skip directive.

#### Protect your secrets

There is actually one element of #[tracing::instrument] that I am not fond of: it automatically attaches all arguments passed to the function to the context of the span - you have to opt-out of logging function inputs (via skip) rather than opt-in6 .

You can prevent this scenario by introducing a wrapper type that explicitly marks which fields are considered
to be sensitive - secrecy::Secret.

There is an additional upside to an explicit wrapper type: it serves as documentation for new developers who are being introduced to the codebase. It nails down what is considered sensitive in your domain/according to the relevant regulation.
The only secret value we need to worry about, right now, is the database password. Let's wrap it:

                use secrecy::Secret; // [..]
                #[derive(serde::Deserialize)] pub struct DatabaseSettings {
                // [...]
                pub password: Secret<String>,
                }

Secret does not interfere with deserialization - Secret implements serde::Deserialize by delegating to the deserialization logic of the wrapped type (if you enable the serde feature flag, as we did).

The compiler error is a great prompt to notice that the entire database connection string should be marked as Secret as well given that it embeds the database password:

#### Request ID

We have one last job to do: ensure all logs for a particular request, in particular the record with the returned status code, are enriched with a request_id property. How?
If our goal is to avoid touching actix_web::Logger the easiest solution is adding another middleware, Re- questIdMiddleware, that is in charge of:
• generatingauniquerequestidentifier;
• creatinganewspanwiththerequestidentifierattachedascontext;
• wrappingtherestofthemiddlewarechaininthenewlycreatedspan.
We would be leaving a lot on the table though: actix_web::Logger does not give us access to its rich inform- ation (status code, processing time, caller IP, etc.) in the same structured JSON format we are getting from other logs - we would have to parse all that information out of its message string.
We are better off, in this case, by bringing in a solution that is tracing-aware.

It is designed as a drop-in replacement of actix-web’s Logger, just based on tracing instead of log:


We have two different request_id for the same request!

We are still generating a request_id at the function-level which overrides the request_id coming from TracingLogger.


It is not an exaggeration to state that tracing is a foundational crate in the Rust ecosystem. While log is the minimum common denominator, tracing is now established as the modern backbone of the whole diagnostics and instrumentation ecosystem.

# February 21, 2023 

## Going Live

### Virtualization: Docker

The easiest way to ensure that our software runs correctly is to tightly control the environment it is being executed into.

what if, instead of shipping code to produc- tion, you could ship a self-contained environment that included your application?!

### Hosting: Digital Ocean

#### Dockerfiles

A Dockerfile is a recipe for your application environment.

They are organised in layers: you start from a base image (usually an OS enriched with a programming language toolchain) and execute a series of commands (COPY, RUN, etc.), one after the other, to build the environment you need.

                # We use the latest Rust stable release as base image
                FROM rust:1.63.0
                # Let's switch our working directory to `app` (equivalent to `cd app`)
                # The `app` folder will be created for us by Docker in case it does not
                # exist already.
                WORKDIR /app
                # Install the required system dependencies for our linking configuration
                RUN apt update && apt install lld clang -y
                # Copy all files from our working environment to our Docker image
                COPY . .
                # Let's build our binary!
                # We'll use the release profile to make it faaaast
                RUN cargo build --release
                # When `docker run` is executed, launch the binary!
                ENTRYPOINT ["./target/release/zero2prod"]

The process of executing those commands to get an image is called building. Using the Docker CLI:

                # Build a docker image tagged as "zero2prod" according to the recipe
                # specified in `Dockerfile`
                docker build --tag zero2prod --file Dockerfile .

#### Build context

docker build generates an image starting from a recipe (the Dockerfile) and a build context.

You can picture the Docker image you are building as its own fully isolated environment.
The only point of contact between the image and your local machine are commands like COPY or ADD2 : the build context determines what files on your host machine are visible inside the Docker container to COPY and its friends.

Using . we are telling Docker to use the current directory as the build context for this image; COPY . app will therefore copy all files from the current directory (including our source code!) into the app directory of our Docker image.

Using . as build context implies, for example, that Docker will not allow COPY to see files from the parent directory or from arbitrary paths on your machine into the image.

#### Sqlx offline mode

Add sqlx offline to cargo.toml

                # Using table-like toml syntax to avoid a super-long line!
                [dependencies.sqlx] version = "0.6" default-features = false features = [
                "runtime-tokio-rustls",
                "macros",
                "postgres",
                "uuid",
                "chrono",
                "migrate",
                "offline"
                ]

sqlx prepare

                sqlx-prepare
                Generate query metadata to support offline compile-time verification.
                Saves metadata for all invocations of `query!` and related macros to
                `sqlx-data.json` in the current directory, overwriting if needed.
                During project compilation, the absence of the `DATABASE_URL` environment
                variable or the presence of `SQLX_OFFLINE` will constrain the compile-time
                verification to only read from the cached query metadata.
                USAGE:
                sqlx prepare [FLAGS] [-- <args>...]
                ARGS:
                <args>...
                        Arguments to be passed to `cargo rustc ...`
                FLAGS:
                        --check
                        Run in 'check' mode. Exits with 0 if the query metadata is up-to-date.
                        Exits with 1 if the query metadata needs updating

In other words, prepare performs the same work that is usually done when cargo build is invoked but it saves the outcome of those queries to a metadata file (sqlx-data.json) which can later be detected by sqlx itself and used to skip the queries altogether and perform an offline build.

                # It must be invoked as a cargo subcommand
                # All options after `--` are passed to cargo itself
                # We need to point it at our library since it contains
                # all our SQL queries.
                cargo sqlx prepare -- --lib

query data written to `sqlx-data.json` in the current directory; please check this into version control

We will indeed commit the file to version control, as the command output suggests.
Let’s set the SQLX_OFFLINE environment variable to true in our Dockerfile to force sqlx to look at the saved metadata instead of trying to query a live database:

                FROM rust:1.63.0
                WORKDIR /app
                RUN apt update && apt install lld clang -y
                COPY . .
                ENV SQLX_OFFLINE true
                RUN cargo build --release
                ENTRYPOINT ["./target/release/zero2prod"]

Then run the build:

                docker build --tag zero2prod --file Dockerfile .

# February 22, 2023 

#### Running an image

When building our image we attached a tag to it, zero2prod:

                docker run zero2prod

docker run will trigger the execution of the command we specified in our ENTRYPOINT statement:

                ENTRYPOINT ["./target/release/zero2prod"]

In our case, it will execute our binary therefore launching our API.


We can relax our requirements by using connect_lazy - it will only try to establish a connection when the pool is used for the first time.


#### Networking

By default, Docker images do not expose their ports to the underlying host machine. We need to do it explicitly using the -p flag.
Let’s kill our running image to launch it again using:

                docker run -p 8000:8000 zero2prod

We are using 127.0.0.1 as our host in address - we are instructing our application to only accept connec- tions coming from the same machine.
However, we are firing a GET request to /health_check from the host machine, which is not seen as local by our Docker image, therefore triggering the Connection refused error we have just seen.
We need to use 0.0.0.0 as host to instruct our application to accept connections from any network interface, not just the local one.
We should be careful though: using 0.0.0.0 significantly increases the “audience” of our application, with some security implications.
The best way forward is to make the host portion of our address configurable - we will keep using 127.0.0.1 for our local development and set it to 0.0.0.0 in our Docker images.

#### Hierarchical configuration

Let’s introduce another struct, ApplicationSettings, to group together all configuration values related to our application address:

                #[derive(serde::Deserialize)] pub struct Settings {
                        pub database: DatabaseSettings,
                        pub application: ApplicationSettings,
                }

                #[derive(serde::Deserialize)] pub struct ApplicationSettings {
                        pub port: u16,
                        pub host: String,
                }

We need to update our configuration.yaml file to match the new structure:

as well as our main.rs, where we will leverage the new configurable host field:

The host is now read from configuration, but how do we use a different value for different environments? We need to make our configuration hierarchical.

Let’s have a look at get_configuration, the function in charge of loading our Settings struct:

We are reading from a file named configuration to populate Settings’s fields. There is no further room for tuning the values specified in our configuration.yaml.
Let’s take a more refined approach. We will have:
        • A base configuration file, for values that are shared across our local and production environment (e.g. database name);
        • A collection of environment-specific configuration files, specifying values for fields that require cus- tomisation on a per-environment basis (e.g. host);
        • An environment variable, APP_ENVIRONMENT, to determine the running environment (e.g. produc- tion or local).
All configuration files will live in the same top-level directory, configuration.
The good news is that config, the crate we are using, supports all the above out of the box!


Let’s refactor our configuration file to match the new structure.
We have to get rid of configuration.yaml and create a new configuration directory with base.yaml, local.yaml and production.yaml inside.

We can now instruct the binary in our Docker image to use the production configuration by setting the APP_ENVIRONMENT environment variable with an ENV instruction:


#### Database connectivity

Let’s fail a little faster by using a shorter timeout:

#### Docker image size

                docker images <name> (here it is zero2prod, the tag we gave it)

can also run it for Rust

                docker images rust:1.63.0

Our first line of attack is reducing the size of the Docker build context by excluding files that are not needed to build our image.
Docker looks for a specific file in our project to determine what should be ignored - .dockerignore
Let’s create one in the root directory with the following content:

                .env
                target/
                tests/
                Dockerfile
                scripts/
                migrations/

All files that match the patterns specified in .dockerignore are not sent by Docker as part of the build context to the image, which means they will not be in scope for COPY instructions.
This will massively speed up our builds (and reduce the size of the final image) if we get to ignore heavy directories (e.g. the target folder for Rust projects).

The next optimisation, instead, leverages one of Rust’s unique strengths.
Rust’s binaries are statically linked3 - we do not need to keep the source code or intermediate compilation artifacts around to run the binary, it is entirely self-contained.
This plays nicely with multi-stage builds, a useful Docker feature. We can split our build in two stages:
        • abuilderstage,togenerateacompiledbinary; 
        • aruntimestage,torunthebinary.

runtime is our final image.
The builder stage does not contribute to its size - it is an intermediate step and it is discarded at the end of the build. The only piece of the builder stage that is found in the final artifact is what we explicitly copy over - the compiled binary!

We can go even smaller by shaving off the weight of the whole Rust toolchain and machinery (i.e. rustc, cargo, etc) - none of that is needed to run our binary.
We can use the bare operating system as base image (debian:bullseye-slim) for our runtime stage:

#### Caching for Rust Docker builds

Each RUN, COPY and ADD instruction in a Dockerfile creates a layer: a diff between the previous state (the layer above) and the current state after having executed the specified command.

Layers are cached: if the starting point of an operation has not changed (e.g. the base image) and the com- mand itself has not changed (e.g. the checksum of the files copied by COPY) Docker does not perform any computation and directly retrieves a copy of the result from the local cache.
Docker layer caching is fast and can be leveraged to massively speed up Docker builds.
The trick is optimising the order of operations in your Dockerfile: anything that refers to files that are chan- ging often (e.g. source code) should appear as late as possible, therefore maximising the likelihood of the previous step being unchanged and allowing Docker to retrieve the result straight from the cache.
The expensive step is usually compilation.
Most programming languages follow the same playbook: you COPY a lock-file of some kind first, build your dependencies, COPY over the rest of your source code and then build your project.
This guarantees that most of the work is cached as long as your dependency tree does not change between one build and the next.


cargo, unfortunately, does not provide a mechanism to build your project dependencies starting from its Cargo.lock file (e.g. cargo build --only-deps).
Once again, we can rely on a community project to expand cargo’s default capability: cargo-chef5 .

We are using three stages: the first computes the recipe file, the second caches our dependencies and then builds our binary, the third is our runtime environment. As long as our dependencies do not change the recipe.json file will stay the same, therefore the outcome of cargo chef cook --release --recipe- path recipe.json will be cached, massively speeding up our builds.
We are taking advantage of how Docker layer caching interacts with multi-stage builds: the COPY . . state- ment in the planner stage will invalidate the cache for the planner container, but it will not invalidate the cache for the builder container as long as the checksum of the recipe.json returned by cargo chef pre- pare does not change.
You can think of each stage as its own Docker image with its own caching - they only interact with each other when using the COPY --from statement.
This will save us a massive amount of time in the next section.

[Potential solution of Docker image is not building.](https://9to5answer.com/error-building-docker-image-39-executor-failed-running-bin-sh-c-apt-get-y-update-39)


### Deploy to Digital Ocean Apps Platform

Install Digital Ocean CLI:

                brew install doctl

Hosting on Digital Ocean’s App Platform is not free - keeping our app and its associated database up and running costs roughly 20.00 USD/month.
I suggest you to destroy the app at the end of each session - it should keep your spend way below 1.00 USD. I spent 0.20 USD while playing around with it to write this chapter!

#### App specification

Digital Ocean’s App Platform uses a declarative configuration file to let us specify what our application deployment should look like - they call it App Spec.
Looking at the reference documentation, as well as some of their examples, we can piece together a first draft of what our App Spec looks like.
Let’s put this manifest, spec.yaml, at the root of our project directory.

We can use their CLI, doctl, to create the application for the first time:

                doctl apps create --spec spec.yaml

Create a token on [Digital Ocean's API page](https://cloud.digitalocean.com/account/api/tokens?i=3a0da8)

Authenticate with

                doctl auth init

Create the application:

                doctl apps create --spec spec.yaml

check the application with (or look on the Digital Ocean apps page):

                doctl apps list

Although the app has been successfully created it is not running yet!
Check the Deployment tab on their dashboard - it is probably building the Docker image.
Looking at a few recent issues on their bug tracker it might take a while - more than a few people have reported they experienced slow builds. Digital Ocean’s support engineers suggested to leverage Docker layer caching to mitigate the issue - we already covered all the bases there!

Try firing off a health check request now, it should come back with a 200 OK!
Notice that DigitalOcean took care for us to set up HTTPS by provisioning a certificate and redirecting HTTPS traffic to the port we specified in the application specification. One less thing to worry about.
The POST /subscriptions endpoint is still failing, in the very same way it did locally: we do not have a live database backing our application in our production environment.
Let’s provision one in the spec.yaml

Then update the app specification

                # You can retrieve your app id using `doctl apps list`
                doctl apps update YOUR-APP-ID --spec=spec.yaml

It will take some time for DigitalOcean to provision a Postgres instance.
In the meantime we need to figure out how to point our application at the database in production.

#### How to inject secrets using environment variables

The connection string will contain values that we do not want to commit to version control - e.g. the user- name and the password of our database root user.
Our best option is to use environment variables as a way to inject secrets at runtime into the application environment. DigitalOcean’s apps, for example, can refer to the DATABASE_URL environment variable (or a few others for a more granular view) to get the database connection string at runtime.

This allows us to customize any value in our Settings struct using environment variables, overriding what is specified in our configuration files.
Why is that convenient?
It makes it possible to inject values that are too dynamic (i.e. not known a priori) or too sensitive to be stored in version control.
It also makes it fast to change the behaviour of our application: we do not have to go through a full re-build if we want to tune one of those values (e.g. the database port). For languages like Rust, where a fresh build can take ten minutes or more, this can make the difference between a short outage and a substantial service degradation with customer-visible impact.

Before we move on let’s take care of an annoying detail: environment variables are strings for the config crate and it will fail to pick up integers if using the standard deserialization routine from serde. Luckily enough, we can specify a custom deserialization function.

Let’s add a new dependency, serde-aux (serde auxiliary) and let’s modify both ApplicationSettings and DatabaseSettings

# February 24, 2023 

#### Connecting to Digital Ocean's Postgres Instance

Let’s have a look at the connection string of our database using DigitalOcean’s dashboard (Components -> Database):

                postgresql://newsletter:<PASSWORD>@<HOST>:<PORT>/newsletter?sslmode=require

Our current DatabaseSettings does not handle SSL mode - it was not relevant for local development, but it is more than desirable to have transport-level encryption for our client/database communication in pro- duction.
Before trying to add new functionality, let’s make room for it by refactoring DatabaseSettings.

We will change its two methods to return a PgConnectOptions instead of a connection string: it will make it easier to manage all these moving parts.

We’ll also have to update src/main.rs and tests/health_check.rs:

Let’s now add the require_ssl property we need to DatabaseSettings:

                #[derive(serde::Deserialize)] pub struct DatabaseSettings {
                        // [...]
                        // Determine if we demand the connection to be encrypted or not
                        pub require_ssl: bool,
                }

We want require_ssl to be false when we run the application locally (and for our test suite), but true in our production environment.

We can take the opportunity - now that we are using PgConnectOptions - to tune sqlx’s instrumentation: lower their logs from INFO to TRACE level.
This will eliminate the noise we noticed in the previous chapter.

#### Environment variables in the App Spec

One last step: we need to amend our spec.yaml manifest to inject the environment variables we need. (in spec.yaml)

The scope is set to RUN_TIME to distinguish between environment variables needed during our Docker build process and those needed when the Docker image is launched.
We are populating the values of the environment variables by interpolating what is exposed by the Digital Ocean’s platform (e.g. ${newsletter.PORT}) - refer to their documentation for more details.

Let's apply the new spec.

                # You can retrieve your app id using `doctl apps list`
                doctl apps update YOUR-APP-ID --spec=spec.yaml

and push our change up to GitHub to trigger a new deployment.

We now need to migrate the database6:

                DATABASE_URL=YOUR-DIGITAL-OCEAN-DB-CONNECTION-STRING sqlx migrate run

Let’s fire off a POST request to /subscriptions:

                curl --request POST \
                --data 'name=le%20guin&email=ursula_le_guin%40gmail.com' \
                https://zero2prod-adqrw.ondigitalocean.app/subscriptions \
                --verbose


### Reject invalid subscribers

#### Requirements -- domain constrants, security constraints

Let’s get to the point - what validation should we perform on names to improve our security posture given the class of threats we identified?
I suggest:
        • Enforcingamaximumlength.WeareusingTEXTastypeforouremailinPostgres,whichisvirtually unbounded - well, until disk storage starts to run out. Names come in all shapes and forms, but 256 characters should be enough for the greatest majority of our users4 - if not, we will politely ask them to enter a nickname.
        • Rejectnamescontainingtroublesomecharacters./()"<>\{}arefairlycommoninURLs,SQLquer- ies and HTML fragments - not as much in names5. Forbidding them raises the complexity bar for SQL injection and phishing attempts.

First implementation can use:

                // An extension trait to provide the `graphemes` method // on `String` and `&str`
                use unicode_segmentation::UnicodeSegmentation;

#### Validation is a leaky cauldron

What we need is a parsing function - a routine that accepts unstructured input and, if a set of conditions holds, returns us a more structured output, an output that structurally guarantees that the invariants we care about hold from that point onwards...by Using Types.

#### Type drive development

Type-driven development is a powerful approach to encode the constraints of a domain we are trying to model inside the type system, leaning on the compiler to make sure they are enforced.
The more expressive the type system of our programming language is, the tighter we can constrain our code to only be able to represent states that are valid in the domain we are working in.

Rust has not invented type-driven development - it has been around for a while, especially in the functional programming communities (Haskell, F#, OCaml, etc.). Rust “just” provides you with a type-system that is expressive enough to leverage many of the design patterns that have been pioneered in those languages in the past decades. The particular pattern we have just shown is often referred to as the “new-type pattern” in the Rust community.

# February 28, 2023 

#### Ownerhip meets invariants

