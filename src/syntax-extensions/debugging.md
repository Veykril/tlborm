# Debugging

`rustc` provides a number of tools to debug general syntax extensions, as well as some more specific ones tailored towards declarative and procedural macros respectively.


Sometimes, it is what the extension *expands to* that proves problematic as you do not usually see the expanded code.
Fortunately `rustc` offers the ability to look at the expanded code via the unstable `-Zunpretty=expanded` argument.
Given the following code:

```rust,ignore
// Shorthand for initializing a `String`.
macro_rules! S {
    ($e:expr) => {String::from($e)};
}

fn main() {
    let world = S!("World");
    println!("Hello, {}!", world);
}
```

compiled with the following command:

```shell
rustc +nightly -Zunpretty=expanded hello.rs
```

produces the following output (modified for formatting):

```rust,ignore
#![feature(prelude_import)]
#[prelude_import]
use std::prelude::rust_2018::*;
#[macro_use]
extern crate std;
// Shorthand for initializing a `String`.
macro_rules! S { ($e : expr) => { String :: from($e) } ; }

fn main() {
    let world = String::from("World");
    {
        ::std::io::_print(
            ::core::fmt::Arguments::new_v1(
                &["Hello, ", "!\n"],
                &match (&world,) {
                    (arg0,) => [
                        ::core::fmt::ArgumentV1::new(arg0, ::core::fmt::Display::fmt)
                    ],
                }
            )
        );
    };
}
```

But not just `rustc` exposes means to aid in debugging syntax extensions.
For the aforementioned `-Zunpretty=expanded` option, there exists a nice `cargo` plugin called [`cargo-expand`](https://github.com/dtolnay/cargo-expand) made by [`dtolnay`](https://github.com/dtolnay) which is basically just a wrapper around it.

You can also use the [playground](https://play.rust-lang.org/), clicking on its `TOOLS` button in the top right gives you the option to expand syntax extensions as well!
