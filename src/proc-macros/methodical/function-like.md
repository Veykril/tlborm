# Function-like

Function-like procedural macros are invoked like declarative macros that is `makro!(…)`.

This type of macro is the simplest of the three though.
It is also the only one which you can't differentiate from declarative macros when solely looking at the invocation.

A simple skeleton of a function-like procedural macro looks like the following:
```rs
use proc_macro::TokenStream;

#[proc_macro]
pub fn tlborm_fn_macro(input: TokenStream) -> TokenStream {
    input
}
```

As one can see this is in fact just a mapping from one [`TokenStream`] to another where the `input` will be the tokens inside of the invocation delimiters, e.g. for an example invocation `foo!(bar)` the input token stream would consist of the `bar` token.
The returned token stream will **replace** the macro invocation.

For this macro type the same placement and expansion rules apply as for declarative macros, that is the macro must output a correct token stream for the invocation location.
Unlike with declarative macros though, function-like procedural macros do not have certain restrictions imposed on their inputs though.
That is the restrictions for what may follow fragment specifiers listed in the [Metavariables and Expansion Redux](../../decl-macros/minutiae/metavar-and-expansion.md) chapter listed is not applicable here, as the procedural macros work on the tokens directly instead of matching them against fragment specifiers or similar.

With that said it is apparent that the procedural counter part to these macros is more powerful as they can arbitrarily modify their input, and produce any output desired as long as its within the bounds of the language syntax.

[`TokenStream`]:https://doc.rust-lang.org/proc_macro/struct.TokenStream.html

Usage example:
```rs
use tlborm_proc::tlborm_attribute;

fn foo() {
    tlborm_attribute!(be quick; time is mana);
}
```
