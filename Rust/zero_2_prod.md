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