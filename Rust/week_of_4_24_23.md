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

you can ask Rocket to automatically validate:

    The type of a dynamic path segment.
    The type of several dynamic path segments.
    The type of incoming body data.
    The types of query strings, forms, and form values.
    The expected incoming or outgoing format of a request.
    Any arbitrary, user-defined security or validation policies.
The route attribute and function signature work in tandem to describe these validations. Rocket's code generation takes care of actually validating the properties. This section describes how to ask Rocket to validate against all of these properties and more.

#### Methods

A Rocket route attribute can be any one of get, put, post, delete, head, patch, or options, each corresponding to the HTTP method to match against. For example, the following attribute will match against POST requests to the root path:

        #[post("/")]

#### Dynamic paths

You can declare path segments as dynamic by using angle brackets around variable names in a route's path. For example, if we want to say Hello! to anything, not just the world, we can declare a route like so:

        #[get("/hello/<name>")]
        fn hello(name: &str) -> String {
            format!("Hello, {}!", name)
        }

If we were to mount the path at the root (.mount("/", routes![hello])), then any request to a path with two non-empty segments, where the first segment is hello, will be dispatched to the hello route. For example, if we were to visit /hello/John, the application would respond with Hello, John!.

Any number of dynamic path segments are allowed. A path segment can be of any type, including your own, as long as the type implements the FromParam trait. We call these types parameter guards. Rocket implements FromParam for many of the standard library types, as well as a few special Rocket types. For the full list of provided implementations, see the FromParam API docs. Here's a more complete route to illustrate varied usage:

        #[get("/hello/<name>/<age>/<cool>")]
        fn hello(name: &str, age: u8, cool: bool) -> String {
            if cool {
                format!("You're a cool {} year old, {}!", age, name)
            } else {
                format!("{}, we need to talk about your coolness.", name)
            }
        }

#### Multiple segments

You can also match against multiple segments by using <param..> in a route path. The type of such parameters, known as segments guards, must implement FromSegments. A segments guard must be the final component of a path: any text after a segments guard will result in a compile-time error.

As an example, the following route matches against all paths that begin with /page:

        use std::path::PathBuf;

        #[get("/page/<path..>")]
        fn get_page(path: PathBuf) { /* ... */ }

The path after /page/ will be available in the path parameter, which may be empty for paths that are simply /page, /page/, /page//, and so on. The FromSegments implementation for PathBuf ensures that path cannot lead to path traversal attacks. With this, a safe and secure static file server can be implemented in just 4 lines:

        use std::path::{Path, PathBuf};
        use rocket::fs::NamedFile;

        #[get("/<file..>")]
        async fn files(file: PathBuf) -> Option<NamedFile> {
            NamedFile::open(Path::new("static/").join(file)).await.ok()
        }

Tip: Rocket makes it even easier to serve static files!
If you need to serve static files from your Rocket application, consider using FileServer, which makes it as simple as:

rocket.mount("/public", FileServer::from("static/"))

"In web development, static files refer to files such as HTML, CSS, JavaScript, images, and other types of files that are served to the client's web browser and do not change in response to user requests. These files are called "static" because they are pre-existing and are not generated dynamically by the server.

In the code block you provided, FileServer is a struct provided by the Rocket web framework that serves static files. The from() method of FileServer takes a directory path as its argument, and mounts the directory at a given route.

In the example code rocket.mount("/public", FileServer::from("static/")), a new FileServer instance is created that serves files from the "static/" directory, and this server is mounted to the "/public" route in the Rocket application. This means that any files in the "static/" directory can be accessed by clients by visiting URLs that start with "/public".

By using FileServer, you can easily serve your static files without having to write additional code to handle file serving. This can make your code simpler and more concise, and can improve the performance of your application by reducing the load on your server."

#### Ignored segments

A component of a route can be fully ignored by using <_>, and multiple components can be ignored by using <_..>. In other words, the wildcard name _ is a dynamic parameter name that ignores that dynamic parameter. An ignored parameter must not appear in the function argument list. A segment declared as <_> matches anything in a single segment while segments declared as <_..> match any number of segments with no conditions.

As an example, the foo_bar route below matches any GET request with a 3-segment URI that starts with /foo/ and ends with /bar. The everything route below matches every GET request.

        #[get("/foo/<_>/bar")]
        fn foo_bar() -> &'static str {
            "Foo _____ bar!"
        }

        #[get("/<_..>")]
        fn everything() -> &'static str {
            "Hey, you're here."
        }

# April 26, 2023 

#### Forwarding

Let's take a closer look at this route attribute and signature pair from a previous example:


        #[get("/hello/<name>/<age>/<cool>")]
        fn hello(name: &str, age: u8, cool: bool) { /* ... */ }

What if cool isn't a bool? Or, what if age isn't a u8? When a parameter type mismatch occurs, Rocket forwards the request to the next matching route, if there is any. This continues until a route doesn't forward the request or there are no remaining routes to try. When there are no remaining routes, a customizable 404 error is returned.

Routes are attempted in increasing rank order. Rocket chooses a default ranking from -12 to -1, detailed in the next section, but a route's rank can also be manually set with the rank attribute. To illustrate, consider the following routes:

        #[get("/user/<id>")]
        fn user(id: usize) { /* ... */ }

        #[get("/user/<id>", rank = 2)]
        fn user_int(id: isize) { /* ... */ }

        #[get("/user/<id>", rank = 3)]
        fn user_str(id: &str) { /* ... */ }

        #[launch]
        fn rocket() -> _ {
            rocket::build().mount("/", routes![user, user_int, user_str])
        }

Notice the rank parameters in user_int and user_str. If we run this application with the routes mounted at the root path, as is done in rocket() above, requests to /user/<id> (such as /user/123, /user/Bob, and so on) will be routed as follows:

The user route matches first. If the string at the <id> position is an unsigned integer, then the user handler is called. If it is not, then the request is forwarded to the next matching route: user_int.

The user_int route matches next. If <id> is a signed integer, user_int is called. Otherwise, the request is forwarded.

The user_str route matches last. Since <id> is always a string, the route always matches. The user_str handler is called.

#### Default ranking

If a rank is not explicitly specified, Rocket assigns a default rank. The default rank prefers static segments over dynamic segments in both paths and queries: the more static a route's path and query are, the higher its precedence.

There are three "colors" to paths and queries:

    static, meaning all components are static
    partial, meaning at least one component is dynamic
    wild, meaning all components are dynamic

Static paths carry more weight than static queries. The same is true for partial and wild paths. This results in the following default ranking table:

#### Request guards

Request guards are one of Rocket's most powerful instruments. As the name might imply, a request guard protects a handler from being called erroneously based on information contained in an incoming request. More specifically, a request guard is a type that represents an arbitrary validation policy. The validation policy is implemented through the FromRequest trait. Every type that implements FromRequest is a request guard.

Request guards appear as inputs to handlers. An arbitrary number of request guards can appear as arguments in a route handler. Rocket will automatically invoke the FromRequest implementation for request guards before calling the handler. Rocket only dispatches requests to a handler when all of its guards pass.

#### Custom guards

You can implement FromRequest for your own types. For instance, to protect a sensitive route from running unless an ApiKey is present in the request headers, you might create an ApiKey type that implements FromRequest and then use it as a request guard:

        #[get("/sensitive")]
        fn sensitive(key: ApiKey) { /* .. */ }

You might also implement FromRequest for an AdminUser type that authenticates an administrator using incoming cookies. Then, any handler with an AdminUser or ApiKey type in its argument list is assured to only be invoked if the appropriate conditions are met. Request guards centralize policies, resulting in a simpler, safer, and more secure applications.

#### Guard transparency

When a request guard type can only be created through its FromRequest implementation, and the type is not Copy, the existence of a request guard value provides a type-level proof that the current request has been validated against an arbitrary policy. This provides powerful means of protecting your application against access-control violations by requiring data accessing methods to witness a proof of authorization via a request guard. We call the notion of using a request guard as a witness guard transparency.

As a concrete example, the following application has a function, health_records, that returns all of the health records in a database. Because health records are sensitive information, they should only be accessible by super users. The SuperUser request guard authenticates and authorizes a super user, and its FromRequest implementation is the only means by which a SuperUser can be constructed. By declaring the health_records function as follows, access control violations against health records are guaranteed to be prevented at compile-time:

        fn health_records(user: &SuperUser) -> Records { /* ... */ }

The reasoning is as follows:

    The health_records function requires an &SuperUser type.
    The only constructor for a SuperUser type is FromRequest.
    Only Rocket can provide an active &Request to construct via FromRequest.
    Thus, there must be a Request authorizing a SuperUser to call health_records.

We recommend leveraging request guard transparency for all data accesses.

#### Forwarding guards

Request guards and forwarding are a powerful combination for enforcing policies. To illustrate, we consider how a simple authorization system might be implemented using these mechanisms.

We start with two request guards:

    User: A regular, authenticated user.

        The FromRequest implementation for User checks that a cookie identifies a user and returns a User value if so. If no user can be authenticated, the guard forwards.

    AdminUser: A user authenticated as an administrator.

        The FromRequest implementation for AdminUser checks that a cookie identifies an administrative user and returns an AdminUser value if so. If no user can be authenticated, the guard forwards.

We now use these two guards in combination with forwarding to implement the following three routes, each leading to an administrative control panel at /admin:

        use rocket::response::Redirect;

        #[get("/login")]
        fn login() -> Template { /* .. */ }

        #[get("/admin")]
        fn admin_panel(admin: AdminUser) -> &'static str {
            "Hello, administrator. This is the admin panel!"
        }

        #[get("/admin", rank = 2)]
        fn admin_panel_user(user: User) -> &'static str {
            "Sorry, you must be an administrator to access this page."
        }

        #[get("/admin", rank = 3)]
        fn admin_panel_redirect() -> Redirect {
            Redirect::to(uri!(login))
        }

The three routes above encode authentication and authorization. The admin_panel route only succeeds if an administrator is logged in. Only then is the admin panel displayed. If the user is not an admin, the AdminUser guard will forward. Since the admin_panel_user route is ranked next highest, it is attempted next. This route succeeds if there is any user signed in, and an authorization failure message is displayed. Finally, if a user isn't signed in, the admin_panel_redirect route is attempted. Since this route has no guards, it always succeeds. The user is redirected to a log in page.

### Cookies

