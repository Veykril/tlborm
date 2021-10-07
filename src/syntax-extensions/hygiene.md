# Hygiene

Hygiene is an important concept for macros, it describes the ability for a macro to work in its own syntax context, not affecting nor being affected by its surroundings.
In other words this means that a syntax extension should be invocable anywhere without interfering with its surrounding context.

In a perfect world all syntax extensions in rust would be fully hygienic, unfortunately this isn't the case, so care should be taken to avoid writing syntax extensions that aren't fully hygienic.
We will go into general hygiene concepts here which will be touched upon in the corresponding hygiene chapters for the different syntax extensions rust has to offer.

&nbsp;

Hygiene mainly affects identifiers and paths emitted by syntax extensions.
In short, if an identifier created by a syntax extension cannot be accessed by the environment where the syntax extension has been invoked it is hygienic in regards to that identifier.
Likewise, if an identifier used in a syntax extension cannot reference something defined outside of a syntax extension it is considered hygienic.

> **Note**: The terms `create` and `use` refer to the position the identifier is in.
> That is the `Foo` in `struct Foo {}` or the `foo` in `let foo = …;` are created in the sense that they introduce something new under the name,
> but the `Foo` in `fn foo(_: Foo) {}` or the `foo` in `foo + 3` are usages in the sense that they are referring to something existing.

This is best shown by example:

Let's assume we have some syntax extension `make_local` that expands to `let local = 0;`, that is it *creates* the identifier `local`.
Then given the following snippet:
```rust,ignore
make_local!();
assert_eq!(local, 0);
```

If the `local` in `assert_eq!(local, 0);` resolves to the local defined by the syntax extension, the syntax extension is not hygienic(at least in regards to local names/bindings).

Now let's assume we have some syntax extension `use_local` that expands to `local = 42;`, that is it makes *use* of the identifier `local`.
Then given the following snippet:
```rust,ignore
let mut local = 0;
use_local!();
```

If the `local` inside of the syntax extension for the given invocation resolves to the local defined before its invocation, the syntax extension is not hygienic either.

This is a rather short introduction to the general concept of hygiene.
It will be explained in more depth in the corresponding [`macro_rules!` `hygiene`] and [proc-macro `hygiene`] chapters, with their specific peculiarities.

[`macro_rules!` `hygiene`]: ../decl-macros/minutiae/hygiene.md
[proc-macro `hygiene`]: ../proc-macros/hygiene.md
