[4/10/23](#april-10-2023)<br>
[4/11/23](#april-11-2023)<br>
[4/12/23](#april-12-2023)<br>


# April 10, 2023 

Getting into the actual coding for Black Hat Rust.
 
#### Command Line Arguments

Taking an input from the command line, and putting the strings into a vector.

    use std::env;
    fn main() {
        let args: Vec<String> = env::args().collect();
    }

Where std::env imports the module env from the standard library and env::args() calls the args function from this module and returns an iterator which can be “collected” into a Vec<String> , a Vector of String objects. A Vector is an array type that can be resized.

#### Errors

Use the error::Error crate for most out of the box errors.

#### Reading Files

BufReader reads and stores data in an internal buffer, and provides efficient access to this buffered data.

The BufReader type implements the std::io::Read trait, so it can be used wherever a Read trait implementation is required. The buffered data can be accessed through the fill_buf method, which returns a slice of bytes that are currently buffered but have not been consumed yet. When the buffered data is consumed, it can be discarded using the consume method.

        use std::fs::File;
        use std::io::{BufReader, Read};

        fn main() -> std::io::Result<()> {
            let file = File::open("example.txt")?;
            let mut reader = BufReader::new(file);

            let mut buffer = [0; 10];
            reader.read_exact(&mut buffer)?;

            println!("{:?}", buffer);

            Ok(())
        }

In this example, BufReader is created by passing a File object to its constructor. The read_exact method is then called on the BufReader object to read exactly 10 bytes from the file into a buffer. The println statement then prints the contents of the buffer. The advantage of using BufReader is that it reads data from the file in chunks, rather than one byte at a time, which can improve performance.

#### Ok(())

You might also have noticed that the last line of our main function does not contain the return keyword. This is because Rust is an expression-oriented language. Expressions evaluate to a value. Their opposites, statements, are instructions that do something and end with a semicolon ( ; ).
So if our program reaches the last line of the main function, the main function will evaluate to Ok(()) , which means: “success: everything went according to the plan”. An equivalent would have been:
    return Ok(()); but not:
    Ok(());
Because here Ok(()); is a statement due to the semicolon, and the main function no longer evaluates to its expected return type: Result .

#### Avoid lifetime annotations

    - Each elided lifetime in a function’s arguments becomes a distinct lifetime parameter.
    - If there is exactly one input lifetime, elided or not, that lifetime is assigned to all elided lifetimes in the return values of that function.
    - If there are multiple input lifetimes, but one of them is &self or &mut self, the lifetime of self is assigned to all elided output lifetimes.

#### Smart pointers

To obtain a mutable, shared pointer, you can use use the interior mutability pattern:

    Interior mutability is a design pattern in Rust that allows you to mutate data even when there are immutable references to that data; normally, this action is disallowed by the borrowing rules. To mutate data, the pattern uses unsafe code inside a data structure to bend Rust’s usual rules that govern mutation and borrowing. 
        use std::cell::{RefCell, RefMut};

    Box<T> for allocating values on the heap
    Rc<T>, a reference counting type that enables multiple ownership
    Ref<T> and RefMut<T>, accessed through RefCell<T>, a type that enforces the borrowing rules at runtime instead of compile time

#### Arc

Unfortunately, Rc<RefCell<T>> cannot be used across threads or in an async context. This is where Arc comes into play, which implements Send and
Sync and thus is safe to share across threads.

### Project maintenance

#### Rust fmt

    rustfmt is a code formatter that allows codebases to have a consistent coding style and avoid nitpicking during code reviews.
    It can be configured using a .rustfmt.toml file: https://rust-lang.github.io/rustfmt. You can use it by calling:
    $ cargo fmt
    In your projects.

#### Clippy

    clippy is a linter for Rust. It will detect code patterns that may lead to errors or are identified by the community as bad style.
    It helps your codebase to be consistent and reduce time spent during code reviews discussing tiny details.
    It can be installed with:
    $ rustup component add clippy
    And used with:
    $ cargo clippy

#### Cargo outdated

    cargo-outdated is a program that helps you to identify your outdated dependencies that can’t be automatically updated with cargo update
    It can be installed as follows:
    $ cargo install -f cargo-outdated The usage is as simple as running
    $ cargo outdated
    In your projects.

#### Cargo audit

    Sometimes, you may not be able to always keep your dependencies to the last version and need to use an old version (due to dependency by another of your dependency...) of a crate. As a professional, you still want to be sure that none of your outdated dependencies contains any known vulnerability.

    cargo-audit is the tool for the job. It can be installed with: $ cargo install -f cargo-audit

    Like other helpers, it’s very simple to use:
    $ cargo audit
        Fetching advisory database from `https://github.com/RustSec/advisory-db.git`
        Loaded 317 security advisories (from /usr/local/cargo/advisory-db)
        Updating crates.io index
        Scanning Cargo.lock for vulnerabilities (144 crate dependencies)

#### other commands
    rustup update
    cargo update
    cargo check

# April 12, 2023


[look for open-to-the-world services and machines.](https://www.shodan.io/)

[certificate transparency logs](https://certificate.transparency.dev/howctworks/)

[certificates for any site](https://crt.sh/) -- "%.kerkour.com"
    A limitation of this technique is its inability to find non-HTTP(S) services (such as email or VPN servers), and wildcard subdomains ( *.kerkour.com , for example) which may obfuscate the actually used subdomains.

Here is a non-exhaustive list of what can be found by crawling subdomains:
    • Code repositories
    • Forgotten subdomain subject to takeover • Admin panels
    • Shared files
    • Storage buckets
    • Email / Chat servers

#### Errors

For libraries, the current good practice is to use the [thiserror](https://crates.io/crates/thiserror) crate.
For programs, the [anyhow](https://crates.io/crates/anyhow) crate is the recommended one. It will prettify errors returned by
the main function.

