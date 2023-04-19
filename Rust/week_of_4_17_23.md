[4/17/23](#april-17-2023)<br>
[4/18/23](#april-18-2023)<br>
[4/19/23](#april-19-2023)<br>


# April 17, 2023 

A bunch of reading about building applications with ChatGPT.

Leaving Black Hat Rust in section 2.6, book is geared more toward experienced devs, lacks a lot of context in what he's trying to teach.

# April 18, 2023 

Discovered a new guide to learning Rust called [Learning Rust 101](https://rust-lang.guide/#about), going to reach out to the author about mentorship.

# April 19, 2023 

Starting the rust wasm tutorial.

install wasm-pack: https://rustwasm.github.io/wasm-pack/installer/

install cargo-generate: cargo install cargo-generate
    install failed, internet suggested I need [open ssl](https://docs.rs/openssl/0.10.23/openssl/), here are the install notes

    A CA file has been bootstrapped using certificates from the system keychain. To add additional certificates, place .pem files in
  /opt/homebrew/etc/openssl@1.1/certs

    and run /opt/homebrew/opt/openssl@1.1/bin/c_rehash

    openssl@1.1 is keg-only, which means it was not symlinked into /opt/homebrew,
    because macOS provides LibreSSL.

    If you need to have openssl@1.1 first in your PATH, run:
    echo 'export PATH="/opt/homebrew/opt/openssl@1.1/bin:$PATH"' >> ~/.zshrc

    For compilers to find openssl@1.1 you may need to set:
    export LDFLAGS="-L/opt/homebrew/opt/openssl@1.1/lib"
    export CPPFLAGS="-I/opt/homebrew/opt/openssl@1.1/include"

cargo-generate repo: https://github.com/cargo-generate/cargo-generate

update npm: npm install npm@latest -g


#### BUILD

We use wasm-pack to orchestrate the following build steps:

Ensure that we have Rust 1.30 or newer and the wasm32-unknown-unknown target installed via rustup,
Compile our Rust sources into a WebAssembly .wasm binary via cargo,
Use wasm-bindgen to generate the JavaScript API for using our Rust-generated WebAssembly.

by running wasm-pack build

#### WEB PAGE

To take our wasm-game-of-life package and use it in a Web page, we use [the create-wasm-app JavaScript project template](https://github.com/rustwasm/create-wasm-app).

run: npm init wasm-app www

#### INSTALL DEPENDENCIES

go into www: cd www

run: npm install

This command only needs to be run once, and will install the webpack JavaScript bundler and its development server.

Note that webpack is not required for working with Rust and WebAssembly, it is just the bundler and development server we've chosen for convenience here. Parcel and Rollup should also support importing WebAssembly as ECMAScript modules. You can also use Rust and WebAssembly without a bundler if you prefer!

#### USING LOCAL PACKAGE

Rather than use the hello-wasm-pack package from npm, we want to use our local wasm-game-of-life package instead. This will allow us to incrementally develop our Game of Life program.

Open up wasm-game-of-life/www/package.json and next to "devDependencies", add the "dependencies" field, including a "wasm-game-of-life": "file:../pkg" entry:


{
  // ...
  "dependencies": {                     // Add this three lines block!
    "wasm-game-of-life": "file:../pkg"
  },
  "devDependencies": {
    //...
  }
}
Next, modify wasm-game-of-life/www/index.js to import wasm-game-of-life instead of the hello-wasm-pack package:

import * as wasm from "wasm-game-of-life";

wasm.greet();
Since we declared a new dependency, we need to install it:

npm install
Our Web page is now ready to be served locally!

#### SERVING LOCALLY

run: npm run start from www directory

Anytime you make changes and want them reflected on http://localhost:8080/, just re-run the wasm-pack build command within the wasm-game-of-life directory.





