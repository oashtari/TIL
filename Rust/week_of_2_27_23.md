[2/27/23](#february-27-2023)<br>
[2/28/23](#february-28-2023)<br>

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

#### Errors as Values -- Result


