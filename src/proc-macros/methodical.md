# A Methodical Introduction

This chapter will introduce Rust's procedural macro system by explaining the system as a whole.

Unlike a [declarative macro](../decl-macros.md), a procedural macro takes the form of a Rust function taking in a token stream(or two) and outputting a token stream.

A proc-macro is at its core just a function exported from a crate with the `proc-macro` [crate type](https://doc.rust-lang.org/reference/linkage.html), so when writing multiple proc macros you can have them all live in one crate.

> **Note**: When using Cargo, to define a `proc-macro` crate you define and set the `lib.proc-macro` key in the `Cargo.toml` to true.
> ```toml
> [lib]
> proc-macro = true
> ```

A `proc-macro` type crate implicitly links to the compiler-provided [proc_macro](https://doc.rust-lang.org/proc_macro/index.html) crate, which contains all the things you need to get going with developing procedural macros.
The two most important types exposed by the crate are the [`TokenStream`](https://doc.rust-lang.org/proc_macro/struct.TokenStream.html), which are the proc-macro variant of the already familiar token trees as well as the [`Span`](https://doc.rust-lang.org/proc_macro/struct.Span.html), which describes a part of source code used primarily for error reporting and hygiene. See the [Hygiene and Spans](./hygiene.md) chapter for more information.

As proc-macros therefore are functions living in a crate, they can be addressed as all the other items in a Rust project.
All thats required to add the crate to the dependency graph of a project and bring the desired item into scope.

> **Note**: Procedural macros invocations still run at the same stage in the compiler expansion-wise as declarative macros, just that they are standalone Rust programs that the compiler compiles, runs, and finally either replaces or appends to.


## Types of procedural macros

With procedural macros, there are actually exist 3 different kinds with each having slightly different properties.
- *function-like* proc-macros which are used to implement `$name ! $arg` invocable macros
- *attribute* proc-macros which are used to implement `#[$arg]` attributes
- *derive* proc-macros which are used to implement a derive, an *input* to a `#[derive(…)]` attribute

At their core, all 3 work almost the same with a few differences in their inputs and output reflected by their function definition.
As mentioned all a procedural macro really is, is a function that maps a token stream so let's take a quick look at each basic definition and their differences.

### *function-like*
```rs
#[proc_macro]
pub fn my_proc_macro(input: TokenStream) -> TokenStream {
    TokenStream::new()
}
```

### *attribute*
```rs
#[proc_macro_attribute]
pub fn my_attribute(input: TokenStream, annotated_item: TokenStream) -> TokenStream {
    TokenStream::new()
}
```

### *derive*
```rs
#[proc_macro_derive(MyDerive)]
pub fn my_derive(annotated_item: TokenStream) -> TokenStream {
    TokenStream::new()
}
```

As shown, the basic structure is the same for each, a public function marked with an attribute defining its procedural macro type returning a `TokenStream`.
Note how the return type is a `TokenStream` and not a result or something else that gives the notion of being fallible.
This does not mean that proc-macros cannot fail though, in fact they have two ways of reporting errors, the first one being to panic and the second to emit a [`compile_error!`](https://doc.rust-lang.org/std/macro.compile_error.html) invocation.
If a proc-macro panics the compiler will catch it and emit the payload as an error coming from the macro invocation.

> **Beware**: The compiler will happily hang on endless loops spun up inside proc-macros causing the compilation of crates using the proc-macro to hang as well.
