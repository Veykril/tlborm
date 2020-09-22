# Debugging

`rustc` provides a number of tools to debug macros. One of the most useful is [`trace_macros!`],
which is a directive to the compiler instructing it to dump every macro invocation prior to
expansion. For example, given the following:

```rust,ignore
# // Note: make sure to use a nightly channel compiler.
#![feature(trace_macros)]

macro_rules! each_tt {
    () => {};
    ($_tt:tt $($rest:tt)*) => {each_tt!($($rest)*);};
}

each_tt!(foo bar baz quux);
trace_macros!(true);
each_tt!(spim wak plee whum);
trace_macros!(false);
each_tt!(trom qlip winp xod);
# 
# fn main() {}
```

The output is:

```text
note: trace_macro
  --> src/main.rs:11:1
   |
11 | each_tt!(spim wak plee whum);
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: expanding `each_tt! { spim wak plee whum }`
   = note: to `each_tt ! (wak plee whum) ;`
   = note: expanding `each_tt! { wak plee whum }`
   = note: to `each_tt ! (plee whum) ;`
   = note: expanding `each_tt! { plee whum }`
   = note: to `each_tt ! (whum) ;`
   = note: expanding `each_tt! { whum }`
   = note: to `each_tt ! () ;`
   = note: expanding `each_tt! {  }`
   = note: to ``
```

This is *particularly* invaluable when debugging deeply recursive macros. You can also enable this
from the command-line by adding `-Z trace-macros` to the compiler command line.

Secondly, there is [`log_syntax!`] which causes the compiler to output all tokens passed to it. For
example, this makes the compiler sing a song:

```rust
# // Note: make sure to use a nightly channel compiler.
#![feature(log_syntax)]

macro_rules! sing {
    () => {};
    ($tt:tt $($rest:tt)*) => {log_syntax!($tt); sing!($($rest)*);};
}

sing! {
    ^ < @ < . @ *
    '\x08' '{' '"' _ # ' '
    - @ '$' && / _ %
    ! ( '\t' @ | = >
    ; '\x08' '\'' + '$' ? '\x7f'
    , # '"' ~ | ) '\x07'
}
# 
# fn main() {}
```

This can be used to do slightly more targeted debugging than [`trace_macros!`].

Sometimes, it is what the macro *expands to* that proves problematic. For this, the `--pretty`
argument to the compiler can be used. Given the following code:

```rust,ignore
// Shorthand for initialising a `String`.
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
rustc -Z unstable-options --pretty expanded hello.rs
```

produces the following output (modified for formatting):

```rust,ignore
#![feature(no_std, prelude_import)]
#![no_std]
#[prelude_import]
use std::prelude::v1::*;
#[macro_use]
extern crate std as std;
// Shorthand for initialising a `String`.
fn main() {
    let world = String::from("World");
    ::std::io::_print(::std::fmt::Arguments::new_v1(
        {
            static __STATIC_FMTSTR: &'static [&'static str]
                = &["Hello, ", "!\n"];
            __STATIC_FMTSTR
        },
        &match (&world,) {
             (__arg0,) => [
                ::std::fmt::ArgumentV1::new(__arg0, ::std::fmt::Display::fmt)
            ],
        }
    ));
}
```

Other options to `--pretty` can be listed using `rustc -Z unstable-options --help -v`; a full list
is not provided since, as implied by the name, any such list would be subject to change at any time.

But not just `rustc` exposes means to aid in debugging macros. For the aforementioned
`--pretty=expanded` option, there exists a nice `cargo` addon called
[`cargo-expand`](https://github.com/dtolnay/cargo-expand) made by [`dtolnay`](https://github.com/dtolnay)
which is basically just a wrapper around it.

You can also use the [playground](https://play.rust-lang.org/), clicking on its `TOOLS` button in
the top right gives you the option to expand macros right there!

Another amazing tool is [`lukaslueg`'s](https://github.com/lukaslueg)
[`macro_railroad`](https://github.com/lukaslueg/macro_railroad), a tool that allows you visualize
and generate syntax diagrams for Rust's `macro-by-example` style macros. It visualizes the accepted
macro's grammar as an automata. 

[`trace_macros!`]:https://doc.rust-lang.org/std/macro.trace_macros.html
[`log_syntax!`]:https://doc.rust-lang.org/std/macro.log_syntax.html