[2/27/23](#february-27-2023)<br>
[2/28/23](#february-28-2023)<br>
[3/01/23](#march-1-2023)<br>
[3/02/23](#march-2-2023)<br>
[3/03/23](#march-3-2023)<br>

# February 27, 2023 

No studying, had to focus on dealing with county tax assessor, and preparing tax documents.

# February 28, 2023 

#### Ownerhip meets invariants

We changed insert_subscriber’s signature, but we have not amended the body to match the new require- ments - let’s do it now.

We have an issue here: we do not have any way to actually access the String value encapsulated inside Sub- scriberName!
We could change SubscriberName’s definition from SubscriberName(String) to SubscriberName(pub String), but we would lose all the nice guarantees we spent the last two sections talking about:
    • other developers would be allowed to bypass parse and build a SubscriberName with an arbitrary string

    • other developers might still choose to build a SubscriberName using parse but they would then have the option to mutate the inner value later to something that does not satisfy anymore the constraints we care about

We can do better - this is the perfect place to take advantage of Rust’s ownership system! Given a field in a struct we can choose to:
    • expose it by value,consuming the struct itself:    
    • expose a mutable reference:
    • expose a shared reference:

inner_mut is not what we are looking for here - the loss of control on our invariants would be equivalent to using SubscriberName(pub String).

Both inner and inner_ref would be suitable, but inner_ref communicates better our intent: give the caller a chance to read the value without the power to mutate it.
Let’s add inner_ref to SubscriberName - we can then amend insert_subscriber to use it:

#### AsRef

While our inner_ref method gets the job done, I am obliged to point out that Rust’s standard library ex-
poses a trait that is designed exactly for this type of usage - AsRef.

When should you implement AsRef<T> for a type?
When the type is similar enough to T that we can use a &self to get a reference to T itself!

To invoke it with our SubscriberName we would have to first call inner_ref and then call do_something_with_a_string_slice:

Nothing too complicated, but it might take you some time to figure out if SubscriberName can give you a &str as well as how, especially if the type comes from a third-party library.
We can make the experience more seamless by changing do_something_with_a_string_slice’s signature:

This pattern is used quite extensively, for example, in the filesystem module in Rust’s standard library - std::fs. Functions like create_dir take an argument of type P constrained to implement AsRef<Path> instead of forcing the user to understand how to convert a String into a Path. Or how to convert a PathBuf into Path. Or an OsString. Or... you got the gist.

There are other little conversion traits like AsRef in that standard library - they provide a shared interface for the whole ecosystem to standardise around. Implementing them for your types suddenly unlocks a great deal of functionality exposed via generic types in the crates already available in the wild.

We will cover some of the other conversion trait later down the line (e.g. From/Into, TryFrom/TryInto).

Let’s remove inner_ref and implement AsRef<str> for SubscriberName:

#### Panics

Panics in Rust are used to deal with unrecoverable errors: failure modes that were not expected or that we have no way to meaningfully recover from. Examples might include the host machine running out of memory or a full disk.

Rust’s panics are not equivalent to exceptions in languages such as Python, C# or Java. Although Rust provides a few utilities to catch (some) panics, it is most definitely not the recommended approach and should be used sparingly.

burntsushi put it down quite neatly in a Reddit thread a few years ago:

    "[...] If your Rust application panics in response to any user input, then the following should be true:
    your application has a bug, whether it be in a library or in the primary application code."

Adopting this viewpoint we can understand what is happening: when our request handler panics actix- web assumes that something horrible happened and immediately drops the worker that was dealing with that panicking request.7
If panics are not the way to go, what should we use to handle recoverable errors?

# March 1, 2023 

#### Errors as Values -- Result

Rust’s primary error handling mechanism is built on top of the Result type:

    pub enum Result<T, E> { 
        Ok(T),
        Err(E), 
    }

Result is used as the return type for fallible operations: if the operation succeeds, Ok(T) is returned; if it fails, you get Err(E).

It tells us that inserting a subscriber in the database is a fallible operation - if all goes as planned, we don’t get anything back (() - the unit type), if something is amiss we will instead receive a sqlx::Error with details about what went wrong (e.g. a connection issue).

Errors as values, combined with Rust’s enums, are awesome building blocks for a robust error handling story.

#### Insightful Assertion Erros: claim

claim provides a fairly comprehensive range of assertions to work with common Rust types - in particular Option and Result.

#### Unit tests

claim needs our type to implement the Debug trait to provide those nice error messages. Let’s add a #[de- rive(Debug)] attribute on top of SubscriberName:

#### Handling a Result, match and ? operator

? was introduced in Rust 1.13 - it is syntactic sugar.
It reduces the amount of visual noise when you are working with fallible functions and you want to “bubble up” failures (e.g. similar enough to re-throwing a caught exception).

It allows us to return early when something fails using a single character instead of a multi-line block. Given that ? triggers an early return using an Err variant, it can only be used within a function that returns
a Result. subscribe does not qualify (yet).

#### The email format

How do we establish if an email address is “valid”?
There are a few Request For Comments (RFC) by the Internet Engineering Task Force (IETF) outlining the expected structure of an email address - RFC 6854, RFC 5322, RFC 2822. We would have to read them, digest the material and then come up with an is_valid_email function that matches the specification. Unless you have a keen interest in understanding the subtle nuances of the email address format, I would suggest you to take a step back: it is quite messy. So messy that even the HTML specification is willfully non-compliant with the RFCs we just linked.
Our best shot is to look for an existing library that has stared long and hard at the problem to provide us with a plug-and-play solution. Luckily enough, there is at least one in the Rust ecosystem - the validator crate!

Our parse method will just delegate all the heavy-lifting to validator::validate_email:

#### Property Based Testing

We could use another approach to test our parsing logic: instead of verifying that a certain set of inputs is correctly parsed, we could build a random generator that produces valid values and check that our parser does not reject them.

In other words, we verify that our implementation displays a certain property - “No valid email address is rejected”.

This approach is often referred to as property-based testing.

If we were working with time, for example, we could repeatedly sample three random integers
    • H, between 0 and 23 (inclusive); 
    • M, between 0 and 59 (inclusive); 
    • S, between 0 and 59 (inclusive);
and verify that H:M:S is always correctly parsed.

Property-based testing significantly increases the range of inputs that we are validating, and therefore our confidence in the correctness of our code, but it does not prove that our parser is correct - it does not exhaust- ively explore the input space (except for tiny ones).

Let’s see what property testing would look like for our SubscriberEmail.

#### How to generate a random test with fake

fake provides generation logic for both primitive data types (integers, floats, strings) and higher-level objects (IP addresses, country codes, etc.) - in particular, emails! Let’s add fake as a development dependency of our project:

Every time we run our test suite, SafeEmail().fake() generates a new random valid email which we then use to test our parsing logic.
This is already a major improvement compared to a hard-coded valid email, but we would have to run our test suite several times to catch an issue with an edge case. A fast-and-dirty solution would be to add a for loop to the test, but, once again, we can use this as an occasion to delve deeper and explore one of the available testing crates designed around property-based testing.

#### Getting start with quickcheck

quickcheck calls prop in a loop with a configurable number of iterations (100 by default): on every iteration, it generates a new Vec<u32> and checks that prop returned true.
If prop returns false, it tries to shrink the generated input to the smallest possible failing example (the shortest failing vector) to help us debug what went wrong.

Everything is built on top of quickcheck’s Arbitrary trait:

We have two methods:
    • arbitrary: given a source of randomness (g) it returns an instance of the type;
    • shrink: it returns a sequence of progressively “smaller” instances of the type to help quickcheck find
    the smallest possible failure case.

Vec<u32> implements Arbitrary, therefore quickcheck knows how to generate random u32 vectors.

We need to create our own type, let’s call it ValidEmailFixture, and implement Arbitrary for it.

If you look at Arbitrary’s trait definition, you’ll notice that shrinking is optional: there is a default imple- mentation (using empty_shrinker) which results in quickcheck outputting the first failure encountered, without trying to make it any smaller or nicer. Therefore we only need to provide an implementation of Arbitrary::arbitrary for our ValidEmailFixture.

Let’s add both quickcheck and quickcheck-macros as development dependencies:

This is an amazing example of the interoperability you gain by sharing key traits across the Rust ecosystem. How do we get fake and quickcheck to play nicely together?
In Arbitrary::arbitrary we get g as input, an argument of type G.
G is constrained by a trait bound, G: quickcheck::Gen, therefore it must implement the Gen trait in quickcheck, where Gen stands for “generator”.
How is Gen defined?

Anything that implements Gen must also implement the RngCore trait from rand-core.
Let’s examine the SafeEmail faker: it implements the Fake trait.
Fake gives us a fake method, which we have already tried out, but it also exposes a fake_with_rng method, where “rng” stands for “random number generator”.
What does fake accept as a valid random number generator?

You read that right - any type that implements the Rng trait from rand, which is automatically implemented by all types implementing RngCore!
We can just pass g from Arbitrary::arbitrary as the random number generator for fake_with_rng and everything just works!
Maybe the maintainers of the two crates are aware of each other, maybe they aren’t, but a community- sanctioned set of traits in rand-core gives us painless interoperability. Pretty sweet!

#### Payload validation -- refactoring with TryFrom

The refactoring gives us a clearer separation of concerns:
    • parse_subscriber takes care of the conversion from our wire format (the url-decoded data collected from a HTML form) to our domain model (NewSubscriber);
    • subscribe remains in charge of generating the HTTP response to the incoming HTTP request.

The Rust standard library provides a few traits to deal with conversions in its std::convert sub-module. That is where AsRef comes from!
Is there any trait there that captures what we are trying to do with parse_subscriber?
AsRef is not a good fit for what we are dealing with here: a fallible conversion between two types which consumes the input value.
We need to look at TryFrom:

Replace T with FormData, Self with NewSubscriber and Self::Error with String - there you have it, the signature of our parse_subscriber function!

We implemented TryFrom, but we are calling .try_into? What is happening there? There is another conversion trait in the standard library, called TryInto:

Its signature mirrors the one of TryFrom - the conversion just goes in the other direction!
If you provide a TryFrom implementation, your type automatically gets the corresponding TryInto imple- mentation, for free.

try_into takes self as first argument, which allows us to do form.0.try_into() instead of going for NewS- ubscriber::try_from(form.0) - matter of taste, if you want.

Generally speaking, what do we gain by implementing TryFrom/TryInto? Nothing shiny, no new functionality - we are “just” making our intent clearer. We are spelling out “This is a type conversion!”.

Why does it matter? It helps others!
When another developer with some Rust exposure jumps in our codebase they will immediately spot the conversion pattern because we are using a trait that they are already familiar with.

 
# March 2, 2023 

# March 3, 2023 

