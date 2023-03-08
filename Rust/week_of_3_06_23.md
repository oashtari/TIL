[3/06/23](#march-6-2023)<br>
[3/07/23](#march-7-2023)<br>
[3/08/23](#march-8-2023)<br>
[3/09/23](#march-9-2023)<br>
[3/10/23](#march-10-2023)<br>
[3/11/23](#march-11-2023)<br>
[3/12/23](#march-12-2023)<br>

# March 6, 2023 

#### Sharing test helpers

The second option takes full advantage of that each file under tests is its own executable - we can create sub-modules scoped to a single test executable!
Let’s create an api folder under tests, with a single main.rs file inside:

        tests/
            api/
                main.rs
            health_check.rs

First, we gain clarity: we are structuring api in the very same way we would structure a binary crate. Less magic - it builds on the same knowledge of the module system you built while working on application code.

We can now add sub-modules in main.rs:

Add three empty files
tests/api/subscriptions.rs.
Time to delete tests/health_check.rs and re-distribute its content:

There are a few positive side-effects to the new structure: - it is recursive.

If tests/api/subscriptions.rs grows too unwieldy, we can turn it into a module, with tests/api/subscriptions/helpers.rs holding subscription-specific test helpers and one or more test files focused on a specific flow or concern; - the implementation details of our helpers function are encapsulated.

It turns out that our tests only need to know about spawn_app and TestApp - no need to expose configure_database or TRACING, we can keep that complexity hidden away in the helpers module; - we have a single test binary.

If you have large test suite with a flat file structure, you’ll soon be building tens of executable every time you run cargo test. While each executable is compiled in parallel, the linking phase is instead entirely sequential! Bundling all your test cases in a single executable reduces the time spent compiling your test suite in CI3.

#### Sharing startup logic

Every time we add a dependency or modify the server constructor, we have at least two places to modify - we have recently gone through the motions with EmailClient. It’s mildly annoying.
More importantly though, the startup logic in our application code is never tested.
As the codebase evolves, they might start to diverge subtly, leading to different behaviour in our tests com- pared to our production environment.
We will first extract the logic out of main and then figure out what hooks we need to leverage the same code paths in our test code.

#### Extracting our startup code

From a structural perspective, our startup logic is a function taking Settings as input and returning an instance of our application as output.

We first perform some binary-specific logic (i.e. telemetry initialisation), then we build a set of configuration values from the supported sources (files + environment variables) and use it to spin up an application. Linear.

Nothing too surprising - we have just moved around the code that was previously living in main.

#### Testing hooks in our startup logic

At a high-level, we have the following phases:
    • Executetest-specificsetup(i.e.initialiseatracingsubscriber);
    • Randomise the configuration to ensure tests do not interfere with each other (i.e. a different logical
    database for each test case);
    • Initialiseexternalresources(e.g.createandmigratethedatabase!);
    • Buildtheapplication;
    • Launch the application as a background task and return a set of resources to interact with it.

It almost works - the approach falls short at the very end: we have no way to retrieve the random address as- signed by the OS to the application and we don’t really know how to build a connection pool to the database, needed to perform assertions on side-effects impacting the persisted state.

Let’s deal with the connection pool first: we can extract the initialisation logic from build into a stand-alone function and invoke it twice.

You’ll have to add a #[derive(Clone)] to all the structs in src/configuration.rs to make the compiler happy, but we are done with the database connection pool.
How do we get the application address instead?
actix_web::dev::Server, the type returned by build, does not allow us to retrieve the application port. We need to do a bit more legwork in our application code - we will wrap actix_web::dev::Server in a new type that holds on to the information we want.

#### Build an API client

All of our integration tests are black-box: we launch our application at the beginning of each test and interact with it using an HTTP client (i.e. reqwest).
As we write tests, we necessarily end up implementing a client for our API.

It gives us a prime opportunity to see what it feels like to interact with the API as a user.
We just need to be careful not to spread the client logic all over the test suite - when the API changes, we don’t want to go through tens of tests to remove a trailing s from the path of an endpoint.

We have the same calling code in each test - we should pull it out and add a helper method to our TestApp struct:

We could add another method for the health check endpoint, but it’s only used once - there is no need right now.

### Zero downtime deployments

### Database migrations // multi-step migrations

We start by keeping the application code stable.
On the database side, we generate a new migration script:

        sqlx migrate add add_status_to_subscriptions

We can now edit the migration script to add status as an optional column to subscriptions:

        ALTER TABLE subscriptions ADD COLUMN status TEXT NULL;

Run the migration against your local database (SKIP_DOCKER=true ./scripts/init_db.sh): we can now run our test suite to make sure that the code works as is even against the new database schema.
It should pass: go ahead and migrate the production database.

#### Start using the new column

status now exists: we can start using it!
To be precise, we can start writing to it: every time a new subscriber is inserted, we will set status to confirmed.
We just need to change our insertion query from

        //! src/routes/subscriptions.rs
        // [...]
        pub async fn insert_subscriber([...]) -> Result<(), sqlx::Error> { sqlx::query!(
                r#"INSERT INTO subscriptions (id, email, name, subscribed_at)
                VALUES ($1, $2, $3, $4)"#,
                // [...]
        )
        // [...]
        }

to

        //! src/routes/subscriptions.rs
        // [...]
        pub async fn insert_subscriber([...]) -> Result<(), sqlx::Error> { sqlx::query!(
                r#"INSERT INTO subscriptions (id, email, name, subscribed_at, status)
                VALUES ($1, $2, $3, $4, 'confirmed')"#,
                // [...]
        )
        // [...]
        }

#### Backfill and mark as Not Null

The latest version of the application ensures that status is populated for all new subscribers.
To mark status as NOT NULL we just need to backfill the value for historical records: we’ll then be free to alter the column.

Let’s generate a new migration script:

        sqlx migrate add make_status_not_null_in_subscriptions

The SQL migration looks like this:

        -- We wrap the whole migration in a transaction to make sure
        -- it succeeds or fails atomically. We will discuss SQL transactions
        -- in more details towards the end of this chapter!
        -- `sqlx` does not do it automatically for us.
        BEGIN;
            -- Backfill `status` for historical entries
            UPDATE subscriptions
                SET status = 'confirmed'
                WHERE status IS NULL;
            -- Make `status` mandatory
            ALTER TABLE subscriptions ALTER COLUMN status SET NOT NULL;
        COMMIT;

We can migrate our local database, run our test suite and then deploy our production database. We made it, we added status as a new mandatory column!

#### A new table

What about subscription_tokens? Do we need three steps there as well?
No, it is much simpler: we add the new table in a migration while the application keeps ignoring it.
We can then deploy a new version of the application that uses it to enable confirmation emails. Let’s generate a new migration script:

        sqlx migrate add create_subscription_tokens_table

The migration is similar to the very first one we wrote to add subscriptions:

        -- Create Subscription Tokens Table
        CREATE TABLE subscription_tokens(
        subscription_token TEXT NOT NULL,
        subscriber_id uuid NOT NULL
            REFERENCES subscriptions (id),
        PRIMARY KEY (subscription_token)
        );

Pay attention to the details here: the subscriber_id column in subscription_tokens is a foreign key. For each row in subscription_tokens there must exist a row in subscriptions whose id field has the same value of subscriber_id, otherwise the insertion fails. This guarantees that all tokens are attached to a legitimate subscriber.

### Sending a confirmation email

We will build the whole feature in a proper test-driven fashion: small steps in a tight red-green-refactor loop.

#### Red test

To write this test we need to enhance our TestApp.
It currently holds our application and a handle to a pool of connections to the database:

We need to spin up a mock server to stand in for Postmark’s API and intercept outgoing requests, just like we did when we built the email client.
Let’s edit spawn_app accordingly:

Notice that, on failure, wiremock gives us a detailed breakdown of what happened: we expected an incoming request, we received none.
Let’s fix that.

#### Green test

To send an email we need to get our hands on an instance of EmailClient.
As part of the work we did when writing the module, we also registered it in the application context:

We can therefore access it in our handler using web::Data, just like we did for pool:

subscribe_sends_a_confirmation_email_for_valid_data now passes, but subscribe_returns_a_200_for_valid_form_data fails:

It is trying to send an email but it is failing because we haven’t setup a mock in that test. Let’s fix it:

#### A static confirmation link

#### Red test

We don’t care (yet) about the link being dynamic or actually meaningful - we just want to be sure that there is something in the body that looks like a link.
We should also have the same link in both the plain text and the HTML version of the email body.

How do we get the body of a request intercepted by wiremock::MockServer?
We can use its received_requests method - it returns a vector of all the requests intercepted by the server as long as request recording was enabled (the default).

We now need to extract links out of it.
The most obvious way forward would be a regular expression. Let’s face it though: regexes are a messy business and it takes a while to get them right.
Once again, we can leverage the work done by the larger Rust ecosystem - let’s add linkify as a development dependency:

We can use linkify to scan text and return an iterator of extracted links.

#### Green test

We need to tweak our request handler again to satisfy the new test case:

#### Refactor

Our request handler is getting a bit busy - there is a lot of code dealing with our confirmation email now. Let’s extract it into a separate function:

subscribe is once again focused on the overall flow, without bothering with details of any of its steps.

#### Pending confirmation

Let’s look at the status for a new subscriber now.
We are currently setting their status to confirmed in POST /subscriptions, while it should be pending_confirmation until they click on the confirmation link.
Time to fix it.

#### Red test

The name is a bit of a lie - it is checking the status code and performing some assertions against the state stored in the database.
Let’s split it into two separate test cases:

We can now modify the second test case to check the status as well.

#### Green test

We can turn it green by touching again our insert query:

We just need to change confirmed into pending_confirmation:

#### Skeleton of GET /subscriptions/confirm

We have done most of the groundwork on POST /subscriptions - time to shift our focus to the other half of the journey, GET /subscriptions/confirm.
We want to build up the skeleton of the endpoint - we need to register the handler against the path in src/startup.rs and reject incoming requests without the required query parameter, subscription_token. This will allow us to then build the happy path without having to write a massive amount of code all at once
- baby steps!

#### Red test

Let’s add a new module to our tests project to host all test cases dealing with the confirmation callback.

#### Green test

Let’s start with a dummy handler that returns 200 OK regardless of the incoming request:

Time to turn that 200 OK in a 400 Bad Request.
We want to ensure that there is a subscription_token query parameter: we can rely on another one actix-web’s extractors - Query.

The Parameters struct defines all the query parameters that we expect to see in the incoming request. It needs to implement serde::Deserialize to enable actix-web to build it from the incoming request path. It is enough to add a function parameter of type web::Query<Parameter> to confirm to instruct actix-web to only call the handler if the extraction was successful. If the extraction failed a 400 Bad Request is auto- matically returned to the caller.

### Connecting the dots

#### Red test

We will behave like a user: we will call POST /subscriptions, we will extract the confirmation link from the outgoing email request (using the linkify machinery we already built) and then call it to confirm our subscription - expecting a 200 OK.
We will not be checking the status from the database (yet) - that is going to be our grand finale.

There is a fair amount of code duplication going on here, but we will take care of it in due time. Our primary focus is getting the test to pass now.

#### Green test

Let’s begin by taking care of that URL issue. It is currently hard-coded in

The domain and the protocol are going to vary according to the environment the application is running into: it will be http://127.0.0.1 for our tests, it should be a proper DNS record with HTTPS when our application is running in production.
The easiest way to get it right is to pass the domain in as a configuration value.
Let’s add a new field to ApplicationSettings:

        Remember to apply the changes to DigitalOcean every time we touch spec.yaml: grab your app identifier via doctl apps list --format ID and then run doctl apps update $APP_ID --spec spec.yaml.

We now need to register the value in the application context - you should be familiar with the process at this point:

We can now access it in the request handler:

The host is correct, but the reqwest::Client in our test is failing to establish a connection. What is going wrong?
If you look closely, you’ll notice port: None - we are sending our request to http://127.0.0.1/subscriptions/confirm without specifying the port our test server is listening on.
The tricky bit, here, is the sequence of events: we pass in the application_url configuration value before spinning up the server, therefore we do not know what port it is going to listen to (given that the port is randomised using 0!).
This is non-issue for production workloads where the DNS domain is enough - we’ll just patch it in the test. Let’s store the application port in its own field within TestApp:

We can then use it in the test logic to edit the confirmation link:

Not the prettiest, but it gets the job done.

We get a 400 Bad Request back because our confirmation link does not have a subscription_token query parameter attached.
Let’s fix it by hard-coding one for the time being:

#### Refactor

The logic to extract the two confirmation links from the outgoing email request is duplicated across two of our tests - we will likely add more that rely on it as we flesh out the remaining bits and pieces of this feature. It makes sense to extract it in its own helper function.

We are adding it as a method on TestApp in order to get access to the application port, which we need to inject into the links.
It could as well have been a free function taking both wiremock::Request and TestApp (or u16) as paramet- ers - a matter of taste.
We can now massively simplify our two test cases:

The intent of those two test cases is much clearer now.

### Subscription Tokens

We are ready to tackle the elephant in the room: we need to start generating subscription tokens.

#### Red test

We will add a new test case which builds on top of the work we just did: instead of asserting against the returned status code we will check the status of the subscriber stored in the database.

#### Green test

To get the previous test case to pass, we hard-coded a subscription token in the confirmation link:

Let’s refactor send_confirmation_email to take the token as a parameter - it will make it easier to add the generation logic upstream.

Our subscription tokens are not passwords: they are single-use and they do not grant access to protected information.6 Weneedthemtobehardenoughtoguesswhilekeepinginmindthattheworst-casescenario is an unwanted newsletter subscription landing in someone’s inbox.

Given our requirements it should be enough to use a cryptographically secure pseudo-random number gen- erator - a CSPRNG, if you are into obscure acronyms.
Every time we need to generate a subscription token we can sample a sufficiently-long sequence of alphanu- meric characters.

To pull it off we need to add rand as a dependency:

Using 25 characters we get roughly ~10^45 possible tokens - it should be more than enough for our use case. To check if a token is valid in GET /subscriptions/confirm we need POST /subscriptions to store the newly minted tokens in the database.

The table we added for this purpose, subscription_tokens, has two columns: subscription_token and subscriber_id.

We are currently generating the subscriber identifier in insert_subscriber but we never return it to the caller:

Let’s refactor insert_subscriber to give us back the identifier:

We can now tie everything together:

We are done on POST /subscriptions, let’s shift to GET /subscription/confirm:

We need to:
    • get a reference to the database pool;
    • retrieve the subscriber id associated with the token (if one exists); 
    • change the subscriber status to confirmed.
Nothing we haven’t done before - let’s get cracking!

### Database transactions

Transactions are a way to group together related operations in a single unit of work.

The database guarantees that all operations within a transaction will succeed or fail together: the database will never be left in a state where the effect of only a subset of the queries in a transaction is visible.

Going back to our example, if we wrap the two INSERT queries in a transaction we now have two possible end states:
• anewsubscriberanditstokenhavebeenpersisted; • nothinghasbeenpersisted.

Much easier to deal with.

#### Transactions in postgres

To start a transaction in Postgres you use a BEGIN statement. All queries after BEGIN are part of the transac- tion.
The transaction is then finalised with a COMMIT statement.
We have actually already used a transaction in one of our migration scripts!

If any of the queries within a transaction fails the database rolls back: all changes performed by previous queries are reverted, the operation is aborted.
You can also explicitly trigger a rollback with the ROLLBACK statement.

#### Transactions in sqlx

Back to the code: how do we leverage transactions in sqlx?
You don’t have to manually write a BEGIN statement: transactions are so central to the usage of relational databases that sqlx provides a dedicated API.
By calling begin on our pool we acquire a connection from the pool and kick off a transaction:

begin, if successful, returns a Transaction struct.
A mutable reference to a Transaction implements sqlx’s Executor trait therefore it can be used to run queries. All queries run using a Transaction as executor become of the transaction.

Let’s pass transaction down to insert_subscriber and store_token instead of pool:

If you run cargo test now you will see something funny: some of our tests are failing!
Why is that happening?

As we discussed, a transaction has to either be committed or rolled back.
Transaction exposes two dedicated methods: Transaction::commit, to persist changes, and Transaction::rollback, to abort the whole operation.
We are not calling either - what happens in that case?

We can look at sqlx’s source code to understand better. In particular, Transaction’s Drop implementation:

self.open is an internal boolean flag attached to the connection used to begin the transaction and run the queries attached to it.
When a transaction is created, using begin, it is set to true until either rollback or commit are called:

In other words: if commit or rollback have not been called before the Transaction object goes out of scope (i.e. Drop is invoked), a rollback command is queued to be executed as soon as an opportunity arises.7 That is why our tests are failing: we are using a transaction but we are not explicitly committing the changes. When the connection goes back into the pool, at the end of our request handler, all changes are rolled back and our test expectations are not met.
We can fix it by adding a one-liner to subscribe:

Go ahead and deploy the application: seeing a feature working in a live environment adds a whole new level of satisfaction!

# March 7, 2023 

## Error Handling

In Chapter 6 we discussed the building blocks of error handling in Rust - Result and the ? operator.
We left many questions unanswered: how do errors fit within the broader architecture of our application? What does a good error look like? Who are errors for? Should we use a library? Which one?
An in-depth analysis of error handling patterns in Rust will be the sole focus of this chapter.

### What is the purpose of errors?

We are trying to insert a row into the subscription_tokens table in order to store a newly-generated token against a subscriber_id.
execute is a fallible operation: we might have a network issue while talking to the database, the row we are trying to insert might violate some table constraints (e.g. uniqueness of the primary key), etc.

#### Internal errors -- enable the caller to react

The caller of execute most likely wants to be informed if a failure occurs - they need to react accordingly, e.g. retry the query or propagate the failure upstream using ?, as in our example.
Rust leverages the type system to communicate that an operation may not succeed: the return type of execute is Result, an enum.

The caller is then forced by the compiler to express how they plan to handle both scenarios - success and failure.
If our only goal was to communicate to the caller that an error happened, we could use a simpler definition for Result:

There would be no need for a generic Error type - we could just check that execute returned the Err variant, e.g.

        let outcome = sqlx::query!(/* ... */) .execute(transaction)
        .await;
        if outcome == ResultSignal::Err { // Do something if it failed }

This works if there is only one failure mode. Truth is, operations can fail in multiple ways and we might want to react differently depending on what happened.
Let’s look at the skeleton of sqlx::Error, the error type for execute:

sqlx::Error is implemented as an enum to allow users to match on the returned error and behave differently depending on the underlying failure mode. For example, you might want to retry a PoolTimedOut while you will probably give up on a ColumnNotFound.

#### Help an operator to troubleshoot

What if an operation has a single failure mode - should we just use () as error type?
Err(()) might be enough for the caller to determine what to do - e.g. return a 500 Internal Server Error
to the user.
But control flow is not the only purpose of errors in an application.
We expect errors to carry enough context about the failure to produce a report for an operator (e.g. the developer) that contains enough details to go and troubleshoot the issue.
What do we mean by report?
In a backend API like ours it will usually be a log event.
In a CLI it could be an error message shown in the terminal when a --verbose flag is used.
The implementation details may vary, the purpose stays the same: help a human understand what is going wrong.

#### Errors at the edge

Just like operators, users expect the API to signal when a failure mode is encountered.
What does a user of our API see when store_token fails? We can find out by looking at the request handler:

They receive an HTTP response with no body and a 500 Internal Server Error status code.
The status code fulfills the same purpose of the error type in store_token: it is a machine-parsable piece of information that the caller (e.g. the browser) can use to determine what to do next (e.g. retry the request assuming it’s a transient failure).
What about the human behind the browser? What are we telling them?
Not much, the response body is empty.
That is actually a good implementation: the user should not have to care about the internals of the API they are calling - they have no mental model of it and no way to determine why it is failing. That’s the realm of the operator.
We are omitting those details by design.

#### Summary

Let’s summarise what we uncovered so far. Errors serve two1 main purposes:
    • Controlflow(i.e.determinewhatdonext);
    • Reporting(e.g.investigate,afterthefact,whatwentwrongon).

We can also distinguish errors based on their location:
    • Internal(i.e.afunctioncallinganotherfunctionwithinourapplication); 
    • Attheedge(i.e.anAPIrequestthatwefailedtofulfill).

Control flow is scripted: all information required to take a decision on what to do next must be accessible to a machine.
We use types (e.g. enum variants), methods and fields for internal errors.
We rely on status codes for errors at the edge.

Error reports, instead, are primarily consumed by humans.
The content has to be tuned depending on the audience.
An operator has access to the internals of the system - they should be provided with as much context as possible on the failure mode.

A user sits outside the boundary of the application2: they should only be given the amount of information
required to adjust their behaviour if necessary (e.g. fix malformed inputs).
We can visualise this mental model using a 2x2 table with Location as columns and Purpose as rows:
                Internal                        At the edge
    Control     Flow Types, methods, fields     Status codes
    Reporting   Logs/traces                     Response body

We will spend the rest of the chapter improving our error handling strategy for each of the cells in the table.

# March 8, 2023 

#### Error reporting for operators

