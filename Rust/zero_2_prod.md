[2/8/23](#february-8-2023)<br>
[2/9/23](#february-9-2023)<br>
[2/10/23](#february-10-2023)<br>
[2/11/23](#february-11-2023)<br>
[2/13/23](#february-13-2023)<br>
[2/14/23](#february-14-2023)<br>
[2/18/23](#february-18-2023)<br>
[2/19/23](#february-19-2023)<br>
[previous week](/React/template_files.md)

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

