# Hygiene

Hygiene is an important concept for macros, it describes the ability for a macro to work in any context, not affecting nor being affected by its surroundings.
In other words this means that a syntax extension should be invocable anywhere without interfering with its surrounding context which is best shown with an example.

Let's assume we have some syntax extension `make_local` that expands to `let local = 0;`, then given the following snippet:
```rust,ignore
make_local!();
assert_eq!(local, 0);
```
For `make_local` to be considered fully hygienic this snippet should not compile.
On the other hand if this example was to compile, the macro couldn't be considered hygienic, as it impacted its surrounding context by introducing a new name into it.

Now let's assume we have some syntax extension `use_local` that expands to `local = 42;`, then given the following snippet:
```rust,ignore
let mut local = 0;
use_local!();
```

In this case for `use_local` to be considered fully hygienic, this snippet again should not compile as otherwise it would be affected by its surrounding context and also affect its surrounding context as well.

This is a rather short introduction to hygiene which will be explained in more depth in the corresponding [`macro_rules!` `hygiene`] and [proc-macro `hygiene`] chapters as there are specifics to each.

> **Aside**: Rust's syntax extension aren't necessarily fully hygienic, as it depends on the kind of syntax extension being used, for which in some cases one can create fully hygienic ones, but not so for others.

[`macro_rules!` `hygiene`]: ../macros/minutiae/hygiene.md
<!-- FIXME -->
[proc-macro `hygiene`]: ../macros/minutiae/hygiene.md
