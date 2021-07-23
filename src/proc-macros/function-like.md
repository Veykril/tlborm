# `custom!(…)`

> **Note**: This chapter assumes basic knowledge of the `proc_macro2`, `quote` and `syn` crates introduced in the [`third-party-crates` chapter](./third-party-crates.md).

Function-like procedural macros are invoked just like declarative macros, the path to to the macro, followed by a bang(!), followed by the input token tree, e.g `makro!(tokentree).
Just from looking at that invocation one cannot differentiate between the two.

This chapter will introduce this type of procedural macro and implement a basic one to give an example of how to build such a macro.

Here is a simple skeleton of such a macro:
```rs
use proc_macro::TokenStream;

#[proc_macro]
pub fn identity(input: TokenStream) -> TokenStream {
    input
}
```

Note the `pub` visibility and `#[proc_macro]` attribute which are both required, the former as the function has to be exported to be visible to other crates while the latter is required to designate this function as being a function-like procedural macro.
This proc-macro is not that interesting though as all it does is replace its invocation with the input.

Let's build a small macro that parses some input, does some transformation on the parsed input and then emits the transformed tokens.
