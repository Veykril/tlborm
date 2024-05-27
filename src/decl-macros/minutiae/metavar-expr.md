# Metavariable Expressions

> *RFC*: [rfcs#1584](https://github.com/rust-lang/rfcs/blob/master/text/3086-macro-metavar-expr.md)\
> *Tracking Issue*: [rust#83527](https://github.com/rust-lang/rust/issues/83527)\
> *Feature*: `#![feature(macro_metavar_expr)]`

> Note: The example code snippets are very bare bones, trying to show off how they work. If you think you got small snippets with proper isolated usage of these expression please submit them!

As mentioned in the [`methodical introduction`](../macros-methodical.md), Rust has special expressions that can be used by macro transcribers to obtain information about metavariables that are otherwise difficult or even impossible to get.
This chapter will introduce them more in-depth together with usage examples.

- [`$$`](#dollar-dollar-)
- [`${count($ident, depth)}`](#countident-depth)
- [`${index(depth)}`](#indexdepth)
- [`${length(depth)}`](#lengthdepth)
- [`${ignore($ident)}`](#ignoreident)

## Dollar Dollar (`$$`)

The `$$` expression expands to a single `$`, making it effectively an escaped `$`.
This enables the ability in writing macros emitting new macros as the former macro won't transcribe metavariables, repetitions and metavariable expressions that have an escaped `$`.

We can see the problem without using `$$` in the following snippet:
```rust,compile_fail
macro_rules! foo {
    () => {
        macro_rules! bar {
            ( $( $any:tt )* ) => { $( $any )* };
            // ^^^^^^^^^^^ error: attempted to repeat an expression containing no syntax variables matched as repeating at this depth
        }
    };
}

foo!();
# fn main() {}
```

The problem is obvious, the transcriber of foo sees a repetition and tries to repeat it when transcribing, but there is no `$any` metavariable in its scope causing it to fail.
With `$$` we can get around this as the transcriber of `foo` will no longer try to do the repetition.[^tt-$]

```rust,ignore
# // This code block marked `ignore` because mdbook can't handle `#![feature(...)]`.
#![feature(macro_metavar_expr)]

macro_rules! foo {
    () => {
        macro_rules! bar {
            ( $$( $$any:tt )* ) => { $$( $$any )* };
        }
    };
}

foo!();
bar!();
# fn main() {}
```

[^tt-$]: Before `$$` occurs, users must resort to a tricky and not so well-known hack to declare nested macros with repetitions
         [via using `$tt` like this](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=9ce18fc79ce17c77d20e74f3c46ee13c).

## `count($ident, depth)`

The `count` metavariable expression expands to the repetition count of the metavariable `$ident` up to the given repetition depth.

- The `$ident` argument must be a declared metavariable in the scope of the rule.
- The `depth` argument must be an integer literal of value less or equal to the maximum repetition depth that the `$ident` metavariable appears in.
- The expression expands to an unsuffixed integer literal token.

The `count($ident)` expression defaults `depth` to the maximum valid depth, making it count the total repetitions for the given metavariable.

```rust,ignore
# // This code block marked `ignore` because mdbook can't handle `#![feature(...)]`.
#![feature(macro_metavar_expr)]

macro_rules! foo {
    ( $( $outer:ident ( $( $inner:ident ),* ) ; )* ) => {
        println!("count(outer, 0): $outer repeats {} times", ${count($outer)});
        println!("count(inner, 0): The $inner repetition repeats {} times in the outer repetition", ${count($inner, 0)});
        println!("count(inner, 1): $inner repeats {} times in the inner repetitions", ${count($inner, 1)});
    };
}

fn main() {
    foo! {
        outer () ;
        outer ( inner , inner ) ;
        outer () ;
        outer ( inner ) ;
    };
}
```

## `index(depth)`

The `index(depth)` metavariable expression expands to the current iteration index of the repetition at the given depth.

- The `depth` argument targets the repetition at `depth` counting outwards from the inner-most repetition where the expression is invoked.
- The expression expands to an unsuffixed integer literal token.

The `index()` expression defaults `depth` to `0`, making it a shorthand for `index(0)`.

```rust,ignore
# // This code block marked `ignore` because mdbook can't handle `#![feature(...)]`.
#![feature(macro_metavar_expr)]

macro_rules! attach_iteration_counts {
    ( $( ( $( $inner:ident ),* ) ; )* ) => {
        ( $(
            $((
                stringify!($inner),
                ${index(1)}, // this targets the outer repetition
                ${index()}  // and this, being an alias for `index(0)` targets the inner repetition
            ),)*
        )* )
    };
}

fn main() {
    let v = attach_iteration_counts! {
        ( hello ) ;
        ( indices , of ) ;
        () ;
        ( these, repetitions ) ;
    };
    println!("{v:?}");
}
```


## `length(depth)`

The `length(depth)` metavariable expression expands to the iteration count of the repetition at the given depth.

- The `depth` argument targets the repetition at `depth` counting outwards from the inner-most repetition where the expression is invoked.
- The expression expands to an unsuffixed integer literal token.

The `length()` expression defaults `depth` to `0`, making it a shorthand for `length(0)`.


```rust,ignore
# // This code block marked `ignore` because mdbook can't handle `#![feature(...)]`.
#![feature(macro_metavar_expr)]

macro_rules! lets_count {
    ( $( $outer:ident ( $( $inner:ident ),* ) ; )* ) => {
        $(
            $(
                println!(
                    "'{}' in inner iteration {}/{} with '{}' in outer iteration {}/{} ",
                    stringify!($inner), ${index()}, ${length()},
                    stringify!($outer), ${index(1)}, ${length(1)},
                );
            )*
        )*
    };
}

fn main() {
    lets_count!(
        many (small , things) ;
        none () ;
        exactly ( one ) ;
    );
}
```

## `ignore($ident)`

The `ignore($ident)` metavariable expression expands to nothing, making it possible to expand something as often as a metavariable repeats without expanding the metavariable.

- The `$ident` argument must be a declared metavariable in the scope of the rule.

```rust,ignore
# // This code block marked `ignore` because mdbook can't handle `#![feature(...)]`.
#![feature(macro_metavar_expr)]

macro_rules! repetition_tuples {
    ( $( ( $( $inner:ident ),* ) ; )* ) => {
        ($(
            $(
                (
                    ${index()},
                    ${index(1)}
                    ${ignore($inner)} // without this metavariable expression, compilation would fail
                ),
            )*
        )*)
    };
}

fn main() {
    let tuple = repetition_tuples!(
        ( one, two ) ;
        () ;
        ( one ) ;
        ( one, two, three ) ;
    );
    println!("{tuple:?}");
}
```
