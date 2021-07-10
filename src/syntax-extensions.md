# Syntax Extensions

Before talking about Rust's different macro systems it is worthwhile to discuss the general mechanism they are built on: *syntax extensions*.

To do that, we must first discuss how Rust source is processed by the compiler, and the general mechanisms on which user-defined macros and proc-macros are built upon.

> **Note**: This book will use the term *syntax extension* from now on when talking about all of rust's different macro kinds in general to reduce potential confusion with the upcoming [declarative macro 2.0](https://github.com/rust-lang/rust/issues/39412) proposal which uses the `macro` keyword.
