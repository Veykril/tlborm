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

This pattern is perhaps the *most powerful* macro parsing technique available, allowing one to parse grammars of significant complexity.
However, it can increase compile times if used excessively, so should be used
with care.

A *TT muncher* is a recursive `macro_rules!` macro that works by incrementally processing its input one step at a time.
At each step, it matches and removes (munches) some sequence of tokens from the start of its input, generates some intermediate output, then recurses on the input tail.

The reason for "TT" in the name specifically is that the unprocessed part of the input is *always* captured as `$($tail:tt)*`.
This is done as a [`tt`] repetition is the only way to *losslessly* capture part of a macro's input.

The only hard restrictions on TT munchers are those imposed on the `macro_rules!` macro system as a whole:

* You can only match against literals and grammar constructs which can be captured by `macro_rules!`.
* You cannot match unbalanced groups.

It is important, however, to keep the macro recursion limit in mind.
`macro_rules!` does not have *any* form of tail recursion elimination or optimization.
It is recommended that, when writing a TT muncher, you make reasonable efforts to keep recursion as limited as possible.
This can be done by adding additional rules to account for variation in the input (as opposed to recursion into an intermediate layer), or by making compromises on the input syntax to make using standard repetitions more tractable.

[`tt`]: ../minutiae/fragment-specifiers.html#tt

## Performance

TT munchers are inherently quadratic.
Consider a TT muncher rule that consumes one token tree and then recursively calls itself on the remaining input.
If it is passed 100 token trees:
- The initial invocation will match against all 100 token trees.
- The first recursive invocation will match against 99 token trees.
- The next recursive invocation will match against 98 token trees.

And so on, down to 1.
This is a classic quadratic pattern, and long inputs can cause macro expansion to blow out compile times.

Try to avoid using TT munchers too much, especially with long inputs.
The default value of the `recursion_limit` attribute is a good sanity check; if you have to exceed it, you might be heading for trouble.

If you have the choice between writing a TT muncher that can be called once to handle multiple things, or a simpler macro that can be called multiple times to handle a single thing, prefer the latter.
For example, you could change a macro that is called like this:
```rust
# macro_rules! f { ($($tt:tt)*) => {} }
f! {
    fn f_u8(x: u32) -> u8;
    fn f_u16(x: u32) -> u16;
    fn f_u32(x: u32) -> u32;
    fn f_u64(x: u64) -> u64;
    fn f_u128(x: u128) -> u128;
}
```
To one that is called like this:
```rust
# macro_rules! f { ($($tt:tt)*) => {} }
f! { fn f_u8(x: u32) -> u8; }
f! { fn f_u16(x: u32) -> u16; }
f! { fn f_u32(x: u32) -> u32; }
f! { fn f_u64(x: u64) -> u64; }
f! { fn f_u128(x: u128) -> u128; }
```
The longer the input, the more likely this will improve compile times.

Also, if a TT muncher macro has many rules, put the most frequently matched
rules as early as possible.
This avoids unnecessary matching failures.
(In fact, this is good advice for any kind of declarative macro, not just TT munchers.)

Finally, if you can write a macro using normal repetition via `*` or `+`, that should be preferred to a TT muncher.
This is most likely if each invocation of the TT muncher would only process one token at a time.
In more complicated cases, there is an advanced technique used within the `quote` crate that can avoid the quadratic behaviour, at the cost of some conceptual complexity.
See [this comment] for details.

[this comment]: https://github.com/dtolnay/quote/blob/31c3be473d0457e29c4f47ab9cff73498ac804a7/src/lib.rs#L664-L746


