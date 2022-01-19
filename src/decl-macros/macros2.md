# Macros 2.0

> *RFC*: [rfcs#1584](https://github.com/rust-lang/rfcs/blob/master/text/1584-macros.md)\
> *Tracking Issue*: [rust#39412](https://github.com/rust-lang/rust/issues/39412)\
> *Feature*: `#![feature(decl_macro)]`

While not yet stable(or rather far from being finished), there is proposal for a new declarative macro system that is supposed to replace `macro_rules!` dubbed declarative macros 2.0, `macro`, `decl_macro` or confusingly also `macros-by-example.

This chapter is only meant to quickly glance over the current state, showing how to use this macro system and where it differs.
Nothing described here is final and may be subject to change at any time.

## Syntax

We'll do a comparison between the `macro` and `macro_rules` syntax for two macros we have implemented in previous chapters:

```rust
#![feature(decl_macro)]

macro_rules! replace_expr_ {
    ($_t:tt $sub:expr) => { $sub }
}
macro replace_expr($_t:tt $sub:expr) {
    $sub
}

macro_rules! count_tts_ {
    () => { 0 };
    ($odd:tt $($a:tt $b:tt)*) => { (count_tts!($($a)*) << 1) | 1 };
    ($($a:tt $even:tt)*) => { count_tts!($($a)*) << 1 };
}
macro count_tts {
    () => { 0 },
    ($odd:tt $($a:tt $b:tt)*) => { (count_tts!($($a)*) << 1) | 1 },
    ($($a:tt $even:tt)*) => { count_tts!($($a)*) << 1 },
}
```

As can be seen, they look very similar, with just a few differences as well as that `macro`s have two different forms.

Let's inspect the `count_tts` macro first, as that one looks more like what we are used to.
As can be seen, it practically looks identical to the `macro_rules` version with two exceptions, it uses the `macro` keyword and the rule separator is a `,` instead of a `;`.

There is a second form to this though, which is a shorthand for macros that only have one rule.
Taking a look at `replace_expr` we can see that in this case we can write the definition in a way that more resembles an ordinary function.
We can write the matcher directly after the name followed by the transcriber, dropping a pair of braces and the `=>` token.
