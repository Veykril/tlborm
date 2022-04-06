# Third-Party Crates

> **Note**: Crates beyond the automatically linked [`proc_macro`] crate are not required to write procedural macros.
> The crates listed here merely make writing them simpler and more concise, while potentially adding to the compilation time of the procedural macro due to added dependencies.

As procedural macros live in a crate they can naturally depend on ([crates.io](https://crates.io/)) crates.
turns out the crate ecosystem has some really helpful crates tailored towards procedural macros that this chapter will quickly go over, most of which will be used in the following chapters to implement the example macros.
As these are merely quick introductions it is advised to look at each crate's documentation for more in-depth information if required.

## [`proc-macro2`]

[`proc-macro2`], the successor of the [`proc_macro`] crate! Or so you might think but that is of course not correct, the name might be a bit misleading.
This crate is actually just a wrapper around the [`proc_macro`] crate serving two specific purposes, taken from the documentation:
- Bring proc-macro-like functionality to other contexts like build.rs and main.rs.
- Make procedural macros unit testable.

As the [`proc_macro`] crate is exclusive to [`proc_macro`] type crates, making them unit testable or accessing them from non-proc macro code is next to impossible.
With that in mind the [`proc-macro2`] crate mimics the original [`proc_macro`] crate's api, acting as a wrapper in proc-macro crates and standing on its own in non-proc-macro crates.
Hence it is advised to build libraries targeting proc-macro code to be built against [`proc-macro2`] instead as that will enable those libraries to be unit testable, which is also the reason why the following listed crates take and emit [`proc-macro2::TokenStream`](https://docs.rs/proc-macro2/1.0.27/proc_macro2/struct.TokenStream.html)s instead.
When a `proc_macro` token stream is required, one can simply `.into()` the `proc-macro2` token stream to get the `proc_macro` version and vice-versa.

Procedural macros using the `proc-macro2` crate will usually import the `proc-macro2::TokenStream` in an aliased form like `use proc-macro2::TokenStream as TokenStream2`.

## [`quote`]

The [`quote`] crate mainly exposes just one macro, the [`quote!`](https://docs.rs/quote/1/quote/macro.quote.html) macro.

This little macro allows you to easily create token streams by writing the actual source out as syntax while also giving you the power of interpolating tokens right into the written syntax.
[Interpolation](https://docs.rs/quote/1/quote/macro.quote.html#interpolation) can be done by using the `#local` syntax where local refers to a local in the current scope.
Likewise `#( #local )*` can be used to interpolate over an iterator of types that implement [`ToTokens`](https://docs.rs/quote/1/quote/trait.ToTokens.html), this works similar to declarative `macro_rules!` repetitions in that they allow a separator as well as extra tokens inside the repetition.

```rs
let name = /* some identifier */;
let exprs = /* an iterator over expressions tokenstreams */;
let expanded = quote! {
    impl SomeTrait for #name { // #name interpolates the name local from above
        fn some_function(&self) -> usize {
            #( #exprs )* // #name interpolates exprs by iterating the iterator
        }
    }
};
```

This a very useful tool when preparing macro output avoiding the need of creating a token stream by inserting tokens one by one.

> **Note**: As stated earlier, this crate makes use of `proc_macro2` and thus the `quote!` macro returns a `proc-macro2::TokenStream`.

## [`syn`](https://docs.rs/syn/*/syn/)

The [`syn`] crate is a parsing library for parsing a stream of Rust tokens into a syntax tree of Rust source code.
It is a very powerful library that makes parsing proc-macro input quite a bit easier, as the [`proc_macro`] crate itself does not expose any kind of parsing capabilities, merely the tokens.
As the library can be a heavy compilation dependency, it makes heavy use of feature gates to allow users to cut it as small as required.

So what does it offer? A bunch of things.

First of all it has definitions and parsing for all standard Rust syntax nodes(when the `full` feature is enabled), as well as a [`DeriveInput`](https://docs.rs/syn/1/syn/struct.DeriveInput.html) type which encapsulates all the information a derive macro gets passed as an input stream as a structured input(requires the `derive` feature, enabled by default). These can be used right out of the box with the [`parse_macro_input!`](https://docs.rs/syn/1/syn/macro.parse_macro_input.html) macro(requires the `parsing` and `proc-macro` features, enabled by default) to parse token streams into these types.

If Rust syntax doesn't cut it, and instead one wishes to parse custom non-Rust syntax the crate also offers a generic [parsing API](https://docs.rs/syn/1/syn/parse/index.html), mainly in the form of the [`Parse`](https://docs.rs/syn/1/syn/parse/trait.Parse.html) trait(requires the `parsing` feature, enabled by default).

Aside from this the types exposed by the library keep location information and spans which allows procedural macros to emit detailed error messages pointing at the macro input at the points of interest.

As this is again a library for procedural macros, it makes use of the `proc_macro2` token streams and spans and as such, conversions may be required.

[`proc_macro`]: https://doc.rust-lang.org/proc_macro/
[`proc-macro2`]: https://docs.rs/proc-macro2/*/proc_macro2/
[`quote`]: https://docs.rs/quote/*/quote/
[`syn`]: https://docs.rs/syn/*/syn/
