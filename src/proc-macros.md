# Procedural Macros

This chapter will introduce Rust's second syntax extension type, *procedural macros*.

Unlike a [declarative macro](./decl-macros.md), a procedural macro takes the form of a rust function taking in a token stream(or two) and outputting a token stream.
This makes writing procedural macros vastly different from declarative ones and comes with its own share of advantages and disadvantages.

As noted in the [syntax extension chapter](./syntax-extensions/ast.md) it is possible to implement all three syntaxes of macro invocations with procedural macros, `#[attr]` attributes, `#[derive(…)]` derives and `foo!()` function-like. Each of these have some slight differences which will be explained in their separate chapters.

All of these still run at the same stage in the compiler expansion-wise as declarative macros.
As mentioned earlier, a procedural macro is just a function that maps a token stream(a stream of token trees) to another token stream.
Thus it is a program the compiler compiles, then executes by feeding the input of the macro invocation, and depending on the type of macro invocation replaces the invocation with the output entirely or appends it.

So how do we go about creating such a proc-macro? By creating a crate of course!
A proc-macro is at its core just a function exported from a crate with the `proc-macro` [crate type](https://doc.rust-lang.org/reference/linkage.html), so when writing multiple proc macros you can have them all live in one crate.

> **Note**: When using Cargo, to define a `proc-macro` crate you define and set the `lib.proc-macro` key in the `Cargo.toml` to true.
> ```toml
> [lib]
> proc-macro = true
> ```

A `proc-macro` type crate implicitly links to the compiler-provided [proc_macro](https://doc.rust-lang.org/proc_macro/index.html) crate, which contains all the things you need to get going with developing procedural macros.
This crate also exposes the [`TokenStream`](https://doc.rust-lang.org/proc_macro/struct.TokenStream.html) mentioned earlier, which will be your macro's input and output type.
This type as it's documentation says is just a stream of token trees as we know them from earlier chapters but encoded as rust types.
Another type of interest is the [`Span`](https://doc.rust-lang.org/proc_macro/struct.Span.html), which describes a part of source code used primarily used for error reporting and hygiene. Each token has an associated span that may be altered freely.

With this knowledge we can take a look at the function signatures of procedural macros, which in case of a function-like macro or derive is `fn(TokenStream) -> TokenStream` and in case of an attribute is `fn(TokenStream, TokenStream) -> TokenStream`.
Note how the return type is a `TokenStream` itself for both and not a result or something else that gives the notion of fallible.
This does not mean that proc-macros cannot fail though, in fact they have two ways of reporting errors, the first one being to panic and the second to emit a [`compile_error!`](https://doc.rust-lang.org/std/macro.compile_error.html) invocation.
If a proc-macro panics the compiler will catch it and emit the payload as an error coming from the macro invocation.

> **Beware**: The compiler will happily hang on endless loops spun up inside proc-macros causing the compilation of crates using the proc-macro to hang as well.


With all of this in mind we can now start implementing procedural macros, but as it turns out the `proc_macro` crate does not offer any parsing capabilities, just access to the token stream which means we are required to do the parsing of said tokens ourselves.
Parsing tokens to input by hand can be quite cumbersome depending on the macro that is being implemented and as such we will make use of the [`syn`](https://docs.rs/syn/*/syn/) crate to make parsing simpler.
We will quickly go over the aforementioned crate as well as some other helpful ones in the next chapter and then finally begin implementing some macros.

Other resources about procedural macros include the the reference [chapter](https://doc.rust-lang.org/reference/procedural-macros.html) which this chapter took heavy inspiration from.
