### February 8, 2023 
- cargo new <project name> creates new projects
#### to speed up linking phase 
- On MacOS, `brew install michaeleisel/zld/zld`
        [target.x86_64-apple-darwin]
        rustflags = ["-C", "link-arg=-fuse-ld=/usr/local/bin/zld"]
        [target.aarch64-apple-darwin]
        rustflags = ["-C", "link-arg=-fuse-ld=/usr/local/bin/zld"]

#### reduce perceived compilation time
        cargo install cargo-watch
    - to run after each code change
        cargo watch -x check
    - to run check, then test, then run
        cargo watch -x check -x test -x run

#### checking test coverage in your code

At the time of writing tarpaulin only supports
x86_64 CPU architectures running Linux.
        
        cargo install cargo-tarpaulin

- to check coverage

        cargo tarpaulin --ignore-tests

## CI

#### Linting

        rustup component add clippy

- run without getting warnings

        cargo clippy -- -D warnings

#### Formatting

        rustup component add rustfmt

- format whole project with 

        cargo fmt

- in our CI pipeline, we'll simply add a formatting step

        cargo fmt -- --check

#### Security vulnerabilities

        cargo install cargo-audit

- once installed, run

        cargo audit

        