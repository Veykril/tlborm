# Hygiene and Spans

This chapter talks about procedural macro [hygiene](../syntax-extensions/hygiene.md) and the type that encodes it, [`Span`](https://doc.rust-lang.org/proc_macro/struct.Span.html).

Every token in a [`TokenStream`](https://doc.rust-lang.org/proc_macro/struct.TokenStream.html) has an associated `Span` holding some additional info.
A span, as its documentation states, is `A region of source code, along with macro expansion information`.
It points into a region of the original source code(important for displaying diagnostics at the correct places) as well as holding the kind of *hygiene* for this location.
The hygiene is relevant mainly for identifiers, as it allows or forbids the identifier from referencing things or being referenced by things defined outside of the invocation.

There are 3 kinds of hygiene(which can be seen by the constructors of the `Span` type):
- [`definition site`](https://doc.rust-lang.org/proc_macro/struct.Span.html#method.def_site)(***unstable***): A span that resolves at the macro definition site. Identifiers with this span will not be able to reference things defined outside or be referenced by things outside of the invocation. This is what one would call "hygienic".
- [`mixed site`](https://doc.rust-lang.org/proc_macro/struct.Span.html#method.mixed_site): A span that has the same hygiene as `macro_rules` declarative macros, that is it may resolve to definition site or call site depending on the type of identifier. See [here](../decl-macros/minutiae/hygiene.md) for more information.
- [`call site`](https://doc.rust-lang.org/proc_macro/struct.Span.html#method.call_site): A span that resolves to the invocation site. Identifiers in this case will behave as if written directly at the call site, that is they freely resolve to things defined outside of the invocation and can be referenced from the outside as well. This is what one would call "unhygienic".
