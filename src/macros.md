# Macros, A Methodical Introduction

This chapter will introduce Rust's [Macro-By-Example][mbe] system: [`macro_rules!`][mbe]. Rather
than trying to cover it based on practical examples, it will instead attempt to give you a complete
and thorough explanation of *how* the system works. As such, this is intended for people who just
want the system as a whole explained, rather than be guided through it.

There is also the [Macros chapter of the Rust Book] which is a more approachable, high-level
explanation, the reference [chapter](https://doc.rust-lang.org/reference/macros-by-example.html)
and the [practical introduction](./macros-practical.html) chapter of this book, which is a guided
implementation of a single macro.

> **Note**: The following pages will refer to `macro_rules!` macros simply by `macro`.


[mbe]: https://doc.rust-lang.org/reference/macros-by-example.html
[Macros chapter of the Rust Book]: https://doc.rust-lang.org/book/ch19-06-macros.html