[2/20/23](#february-20-2023)<br>
[2/21/23](#february-21-2023)<br>
[2/22/23](#february-22-2023)<br>
[2/24/23](#february-24-2023)<br>

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