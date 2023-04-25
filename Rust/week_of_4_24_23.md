[4/24/23](#april-24-2023)<br>
[4/25/23](#april-25-2023)<br>
[4/26/23](#april-26-2023)<br>
[4/27/23](#april-27-2023)<br>
[4/28/23](#april-28-2023)<br>


# April 24, 2023 

Some reading about various Rust web frameworks.

# April 25, 2023 

Back to the Rocket starting guide.

#### Routing

Rocket applications are centered around routes and handlers. A route is a combination of:

    A set of parameters to match an incoming request against.
    A handler to process the request and return a response.

A handler is simply a function that takes an arbitrary number of arguments and returns any arbitrary type.

        #[get("/world")]              // <- route attribute
        fn world() -> &'static str {  // <- request handler
            "hello, world!"
        }

