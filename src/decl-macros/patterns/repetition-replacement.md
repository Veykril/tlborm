# Repetition Replacement

```rust,ignore
macro_rules! replace_expr {
    ($_t:tt $sub:expr) => {$sub};
}
```

This pattern is where a matched repetition sequence is simply discarded, with the variable being used to instead drive some repeated pattern that is related to the input only in terms of length.

For example, consider constructing a default instance of a tuple with more than 12 elements (the limit as of Rust 1.2).

```rust
macro_rules! tuple_default {
    ($($tup_tys:ty),*) => {
        (
            $(
                replace_expr!(
                    ($tup_tys)
                    Default::default()
                ),
            )*
        )
    };
}
#
# macro_rules! replace_expr {
#     ($_t:tt $sub:expr) => {$sub};
# }
#
# assert_eq!(tuple_default!(i32, bool, String), (i32::default(), bool::default(), String::default()));
```

> **<abbr title="Just for this example">JFTE</abbr>**: we *could* have simply used `$tup_tys::default()`.

Here, we are not actually *using* the matched types.
Instead, we throw them away and replace them with a single, repeated expression.
To put it another way, we don't care *what* the types are, only *how many* there are.
