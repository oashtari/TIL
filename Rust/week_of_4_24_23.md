[4/24/23](#april-24-2023)<br>
[4/25/23](#april-25-2023)<br>
[4/26/23](#april-26-2023)<br>
[4/27/23](#april-27-2023)<br>
[4/28/23](#april-28-2023)<br>


# April 24, 2023 

Some reading about various Rust web frameworks.

# April 25, 2023 

Back to the Rocket starting guide.

## Overview

#### Routing

Rocket applications are centered around routes and handlers. A route is a combination of:

    A set of parameters to match an incoming request against.
    A handler to process the request and return a response.

A handler is simply a function that takes an arbitrary number of arguments and returns any arbitrary type.

        #[get("/world")]              // <- route attribute
        fn world() -> &'static str {  // <- request handler
            "hello, world!"
        }

#### Mounting

Before Rocket can dispatch requests to a route, the route needs to be mounted:

        rocket::build().mount("/hello", routes![world]);

The mount method takes as input:

    A base path to namespace a list of routes under, here, /hello.
    A list of routes via the routes! macro: here, routes![world], with multiple routes: routes![a, b, c].

This creates a new Rocket instance via the build function and mounts the world route to the /hello base path, making Rocket aware of the route. GET requests to /hello/world will be directed to the world function.

Can be chained:

        rocket::build()
            .mount("/hello", routes![world])
            .mount("/hi", routes![world]);

By mounting world to both /hello and /hi, requests to "/hello/world" and "/hi/world" will be directed to the world function.

    Note: In many cases, the base path will simply be "/".

#### Launching

Rocket begins serving requests after being launched, which starts a multi-threaded asynchronous server and dispatches requests to matching routes as they arrive.

There are two mechanisms by which a Rocket can be launched. The first and preferred approach is via the #[launch] route attribute, which generates a main function that sets up an async runtime and starts the server. With #[launch], our complete Hello, world! application looks like:

        #[macro_use] extern crate rocket;

        #[get("/world")]
        fn world() -> &'static str {
            "Hello, world!"
        }

        #[launch]
        fn rocket() -> _ {
            rocket::build().mount("/hello", routes![world])
        }

Tip: #[launch] infers the return type!
Special to Rocket's #[launch] attribute, the return type of a function decorated with #[launch] is automatically inferred when the return type is set to _. If you prefer, you can also set the return type explicitly to Rocket<Build>.

The second approach uses the #[rocket::main] route attribute. #[rocket::main] also generates a main function that sets up an async runtime but unlike #[launch], allows you to start the server:

        #[rocket::main]
        async fn main() -> Result<(), rocket::Error> {
            let _rocket = rocket::build()
                .mount("/hello", routes![world])
                .launch()
                .await?;

            Ok(())
        }

#[rocket::main] is useful when a handle to the Future returned by launch() is desired, or when the return value of launch() is to be inspected. The error handling example for instance, inspects the return value.

#### Futures and Async

Rocket uses Rust Futures for concurrency. Asynchronous programming with Futures and async/await allows route handlers to perform wait-heavy I/O such as filesystem and network access while still allowing other requests to make progress. For an overview of Rust Futures, see Asynchronous Programming in Rust.

In general, you should prefer to use async-ready libraries instead of synchronous equivalents inside Rocket applications.

async appears in several places in Rocket:

    Routes and Error Catchers can be async fns. Inside an async fn, you can .await Futures from Rocket or other libraries.
    Several of Rocket's traits, such as FromData and FromRequest, have methods that return Futures.
    Data and DataStream, incoming request data, and Response and Body, outgoing response data, are based on tokio::io::AsyncRead instead of std::io::Read.

You can find async-ready libraries on crates.io with the async tag.

    Note
    Rocket v0.5 uses the tokio runtime. The runtime is started for you if you use #[launch] or #[rocket::main], but you can still launch() a Rocket instance on a custom-built runtime by not using either attribute.

#### Async routes

        use rocket::tokio::time::{sleep, Duration};

        #[get("/delay/<seconds>")]
        async fn delay(seconds: u64) -> String {
            sleep(Duration::from_secs(seconds)).await;
            format!("Waited for {} seconds", seconds)
        }

First, notice that the route function is an async fn. This enables the use of await inside the handler. sleep is an asynchronous function, so we must await it.

#### Multitasking

Rust's Futures are a form of cooperative multitasking. In general, Futures and async fns should only .await on operations and never block. Some common examples of blocking include locking non-async mutexes, joining threads, or using non-async library functions (including those in std) that perform I/O.

If a Future or async fn blocks the thread, inefficient resource usage, stalls, or sometimes even deadlocks can occur.

Sometimes there is no good async alternative for a library or operation. If necessary, you can convert a synchronous operation to an async one with tokio::task::spawn_blocking:

        use std::io;

        use rocket::tokio::task::spawn_blocking;

        #[get("/blocking_task")]
        async fn blocking_task() -> io::Result<Vec<u8>> {
            // In a real app, use rocket::fs::NamedFile or tokio::fs::File.
            let vec = spawn_blocking(|| std::fs::read("data.txt")).await
                .map_err(|e| io::Error::new(io::ErrorKind::Interrupted, e))??;

            Ok(vec)
        }

## Requests

