# Incremental TT Munchers

```rust
macro_rules! mixed_rules {
    () => {};
    (trace $name:ident; $($tail:tt)*) => {
        {
            println!(concat!(stringify!($name), " = {:?}"), $name);
            mixed_rules!($($tail)*);
        }
    };
    (trace $name:ident = $init:expr; $($tail:tt)*) => {
        {
            let $name = $init;
            println!(concat!(stringify!($name), " = {:?}"), $name);
            mixed_rules!($($tail)*);
        }
    };
}
# 
# fn main() {
#     let a = 42;
#     let b = "Ho-dee-oh-di-oh-di-oh!";
#     let c = (false, 2, 'c');
#     mixed_rules!(
#         trace a;
#         trace b;
#         trace c;
#         trace b = "They took her where they put the crazies.";
#         trace b;
#     );
# }
```

This pattern is perhaps the *most powerful* macro parsing technique available, allowing one to parse
grammars of significant complexity.

A `TT muncher` is a recursive macro that works by incrementally processing its input one step at a
time. At each step, it matches and removes (munches) some sequence of tokens from the start of its
input, generates some intermediate output, then recurses on the input tail.

The reason for "TT" in the name specifically is that the unprocessed part of the input is *always*
captured as `$($tail:tt)*`. This is done as a `tt` repetition is the only way to *losslessly*
capture part of a macro's input.

The only hard restrictions on TT munchers are those imposed on the macro system as a whole:

* You can only match against literals and grammar constructs which can be captured by `macro_rules!`.
* You cannot match unbalanced groups.

It is important, however, to keep the macro recursion limit in mind. `macro_rules!` does not have
*any* form of tail recursion elimination or optimisation. It is recommended that, when writing a TT
muncher, you make reasonable efforts to keep recursion as limited as possible. This can be done by
adding additional rules to account for variation in the input (as opposed to recursion into an
intermediate layer), or by making compromises on the input syntax to make using standard repetitions
more tractable.
