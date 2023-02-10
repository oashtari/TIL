[2/8/23](#february-8-2023)<br>
[2/9/23](#february-9-2023)<br>
[2/10/23](#february-10-2023)<br>


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

FAIL

# February 10 2023 

