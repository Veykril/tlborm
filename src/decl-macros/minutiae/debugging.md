# Debugging

> **Note**: This is a list of debugging tools specifically tailored towards declarative macros, additional means of debugging these can be found in the [debugging chapter](../../syntax-extensions/debugging.md) of syntax extensions.

One of the most useful is [`trace_macros!`], which is a directive to the compiler instructing it to dump every `macro_rules!` macro invocation prior to expansion.
For example, given the following:

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

This is *particularly* invaluable when debugging deeply recursive `macro_rules!` macros.
You can also enable this from the command-line by adding `-Z trace-macros` to the compiler command line.

Secondly, there is [`log_syntax!`] which causes the compiler to output all tokens passed to it.
For example, this makes the compiler sing a song:

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

Another amazing tool is [`lukaslueg`'s](https://github.com/lukaslueg) [`macro_railroad`](https://github.com/lukaslueg/macro_railroad), a tool that allows you visualize and generate syntax diagrams for Rust's `macro_rules!` macros.
It visualizes the accepted macro's grammar as an automata.

[`trace_macros!`]:https://doc.rust-lang.org/std/macro.trace_macros.html
[`log_syntax!`]:https://doc.rust-lang.org/std/macro.log_syntax.html
