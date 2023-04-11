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

### Mental Models for Approaching Rust

