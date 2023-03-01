[2/13/23](#february-13-2023)<br>
[2/14/23](#february-14-2023)<br>
[2/18/23](#february-18-2023)<br>
[2/19/23](#february-19-2023)<br>

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