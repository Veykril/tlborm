# Function-like

Function-like procedural macros are invoked just like declarative macros, the path to to the macro, followed by a bang(!), followed by the input token tree, e.g `makro!(tokentree).
This type of macro is the simplest of the three though it is also the only one which you can't differentiate from declarative macros when solely looking at the invocation.

As was shown in the chapter introduction, a simple skeleton of a function-like procedural macro looks like the following:
```rs
use proc_macro::TokenStream;

#[proc_macro]
pub fn identity(input: TokenStream) -> TokenStream {
    input
}
```

As one can see this is in fact just a mapping from one [`TokenStream`] to another where the `input` will be the tokens inside of the invocation delimiters, e.g. for an example invocation `foo!(bar)` the input token stream would consist of the `bar` token.
When invoked the invocation will simply be replaced with the output [`TokenStream`], or error should the expansion fail which may be due to the macro panicking or producing an invalid token stream for the invocation position.

For this macro type the same placement and expansion rules apply as for declarative macros, that is the macro must output a correct token stream for the invocation location.
Unlike with declarative macros though, function-like procedural macros do not have certain restrictions imposed on their inputs though.
That is the restrictions for what may follow fragment specifiers listed in the [Metavariables and Expansion Redux](../../decl-macros/minutiae/metavar-and-expansion.md) chapter listed is not applicable here, as the procedural macros work on the tokens directly instead of matching them against fragment specifiers or similar.

With that said it is apparent that the procedural counter part to these macros is more powerful as they can arbitrarily modify their input, and produce any output desired as long as its within the bounds of the language syntax.

[`TokenStream`]:https://doc.rust-lang.org/proc_macro/struct.TokenStream.html
