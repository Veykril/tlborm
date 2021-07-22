# Declarative Macros

This chapter will introduce Rust's declarative macro system: [`macro_rules!`][mbe].

There are two different introductions in this chapter, a [methodical] and a [practical].

The former will attempt to give you a complete and thorough explanation of *how* the system works, while the latter one will cover more practical examples.
As such, the [methodical introduction][methodical] is intended for people who just want the system as a whole explained, while the [practical introduction][practical] guides one through the implementation of a single macro.

Following up the two introductions it offers some generally very useful [patterns] and [building blocks] for creating feature rich macros.

Other resources about declarative macros include the [Macros chapter of the Rust Book] which is a more approachable, high-level explanation as well as the reference [chapter](https://doc.rust-lang.org/reference/macros-by-example.html) which goes more into the precise details of things.

> **Note**: This book will usually use the term *mbe*(**M**acro-**B**y-**E**xample), *mbe macro* or `macro_rules!` macro when talking about `macro_rules!` macros.

[mbe]: https://doc.rust-lang.org/reference/macros-by-example.html
[Macros chapter of the Rust Book]: https://doc.rust-lang.org/book/ch19-06-macros.html
[practical]: ./decl-macros/macros-practical.md
[methodical]: ./decl-macros/macros-methodical.md
[patterns]: ./decl-macros/patterns.md
[building blocks]: ./decl-macros/building-blocks.md
