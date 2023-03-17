[3/13/23](#march-13-2023)<br>
[3/14/23](#march-14-2023)<br>
[3/15/23](#march-15-2023)<br>
[3/16/23](#march-16-2023)<br>
[3/17/23](#march-17-2023)<br>
[3/18/23](#march-18-2023)<br>
[3/19/23](#march-19-2023)<br>

# March 13, 2023 

I was stuck :(


# March 14, 2023 

FINALLY, got help with resolving the bug.

# March 15, 2023 

rest day

# March 16, 2023 

## Error Handling

### Internal errors

#### Enable the caller to react

The caller of execute most likely wants to be informed if a failure occurs - they need to react accordingly, e.g. retry the query or propagate the failure upstream using ?, as in our example.

Rust leverages the type system to communicate that an operation may not succeed: the return type of execute is Result, an enum.

The caller is then forced by the compiler to express how they plan to handle both scenarios - success and failure.

Truth is, operations can fail in multiple ways and we might want to react differently depending on what happened.
Let’s look at the skeleton of sqlx::Error, the error type for execute:

sqlx::Error is implemented as an enum to allow users to match on the returned error and behave differently depending on the underlying failure mode. For example, you might want to retry a PoolTimedOut while you will probably give up on a ColumnNotFound.

#### Help an operator to troubleshoot

But control flow is not the only purpose of errors in an application.
We expect errors to carry enough context about the failure to produce a report for an operator (e.g. the developer) that contains enough details to go and troubleshoot the issue.

What do we mean by report?
In a backend API like ours it will usually be a log event.
In a CLI it could be an error message shown in the terminal when a --verbose flag is used.
The implementation details may vary, the purpose stays the same: help a human understand what is going wrong.

#### Errors at the edge

What about the human behind the browser? What are we telling them?
Not much, the response body is empty.
That is actually a good implementation: the user should not have to care about the internals of the API they are calling - they have no mental model of it and no way to determine why it is failing. That’s the realm of the operator.
We are omitting those details by design.

It is, I must admit, not the most useful error message: we are telling the user that the email address they entered is wrong, but we are not helping them to determine why.
In the end, it doesn’t matter: we are not sending any of that information to the user as part of the response of the API - they are getting a 400 Bad Request with no body.

This is a poor error: the user is left in the dark and cannot adapt their behaviour as required.

#### Summary

Errors serve two main purposes:
    • Control flow (i.e. determine what do next);
    • Reporting (e.g. investigate, after the fact, what went wrong on).
We can also distinguish errors based on their location:
    • Internal (i.e. a function calling another function within our application); 
    • At the edge (i.e. an API request that we failed to fulfill).

Control flow is scripted: all information required to take a decision on what to do next must be accessible to a machine.
We use types (e.g. enum variants), methods and fields for internal errors.
We rely on status codes for errors at the edge.
Error reports, instead, are primarily consumed by humans.
The content has to be tuned depending on the audience.
An operator has access to the internals of the system - they should be provided with as much context as possible on the failure mode.

A user sits outside the boundary of the application2: they should only be given the amount of information
required to adjust their behaviour if necessary (e.g. fix malformed inputs).
We can visualise this mental model using a 2x2 table with Location as columns and Purpose as rows:
                    Internal                At the edge
Control Flow        Types, methods, fields  Status codes
Reporting           Logs/traces             Response body

### Error reporting for operators



# March 17, 2023 
# March 18, 2023 
# March 19, 2023 
