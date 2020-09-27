# Syntax Extensions

Before talking about `macro_rules!`, it is worthwhile to discuss the general mechanism they are built on:
syntax extensions. To do that, we must discuss how Rust source is processed by the compiler, and the
general mechanisms on which user-defined `macro_rules!` macros are built.