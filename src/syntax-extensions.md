# Syntax Extensions

<!-- Before talking about macros and proc-macros, we will have to establish some terminology first.
As rust has a few different kinds of macros we will address any kind of macro when not talking about one kind specifically with `syntax extension`.
The reason for that is that rust has `proc macros`, `macro_rules! macros` and `macro 2.0 macros`.
Due to the latter using the `macro` keyword for its definition, using just `macro` as a general term might become confusing in the future hence the use of a different term. -->

Before talking about Rust's many macros it is worthwhile to discuss the general mechanism they are built on: syntax extensions.
To do that, we must discuss how Rust source is processed by the compiler, and the general mechanisms on which user-defined macros and proc-macros are built upon.
