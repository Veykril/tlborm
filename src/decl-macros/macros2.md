# Macros 2.0

> *RFC*: [rfcs#1584](https://github.com/rust-lang/rfcs/blob/master/text/1584-macros.md)\
> *Tracking Issue*: [rust#39412](https://github.com/rust-lang/rust/issues/39412)\
> *Feature*: `#![feature(decl_macro)]`

While not yet stable(or rather far from being finished), there is proposal for a new declarative macro system that is supposed to replace `macro_rules!` dubbed declarative macros 2.0, `macro`, `decl_macro` or confusingly also `macros-by-example`.

This chapter is only meant to quickly glance over the current state, showing how to use this macro system and where it differs.
Nothing described here is final or complete, and may be subject to change.

## Syntax

We'll do a comparison between the `macro` and `macro_rules` syntax for two macros we have implemented in previous chapters:

```rust,ignore
# // This code block marked `ignore` because mdbook can't handle `#![feature(...)]`.
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

Syntax for invoking `macro`s is the same as for `macro_rules` and function-like procedural macros, the name followed by a `!` followed by the macro input token tree.

## `macro` are proper items

Unlike with `macro_rules` macros, which are textually scoped and require `#[macro_export]`(and potentially a re-export) to be treated as an item, `macro` macros behave like proper rust items by default.

As such, you can properly qualify them with visibility specifiers like `pub`, `pub(crate)`, `pub(in path)` and the like.


## Hygiene

Hygiene is by far the biggest difference between the two declarative macro systems.
Unlike `macro_rules` which have [mixed site hygiene], `macro` have definition site hygiene, meaning they do not leak identifiers outside of their invocation.

As such the following compiles with a `macro_rules` macro, but fails with a `macro` definition:

```rust,ignore
# // This code block marked `ignore` because mdbook can't handle `#![feature(...)]`.
#![feature(decl_macro)]
// try uncommenting the following line, and commenting out the line right after

macro_rules! foo {
// macro foo {
    ($name: ident) => {
        pub struct $name;

        impl $name {
            pub fn new() -> $name {
                $name
            }
        }
    }
}

foo!(Foo);

fn main() {
    // this fails with a `macro`, but succeeds with a `macro_rules`
    let foo = Foo::new();
}
```

There may be plans to allow escaping hygiene for identifiers(hygiene bending) in the future.


[mixed site hygiene]: ./minutiae/hygiene.md
