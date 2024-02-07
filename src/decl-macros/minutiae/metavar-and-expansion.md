# Metavariables and Expansion Redux

Once the parser begins consuming tokens for a metavariable, *it cannot stop or backtrack*.
This means that the second rule of the following macro *cannot ever match*, no matter what input is provided:

```ignore
macro_rules! dead_rule {
    ($e:expr) => { ... };
    ($i:ident +) => { ... };
}
```

Consider what happens if this macro is invoked as `dead_rule!(x+)`.
The interpreter will start at the first rule, and attempt to parse the input as an expression.
The first token `x` is valid as an expression.
The second token is *also* valid in an expression, forming a binary addition node.

At this point, given that there is no right-hand side of the addition, you might expect the parser to give up and try the next rule.
Instead, the parser will panic and abort the entire compilation, citing a syntax error.

As such, it is important in general that you write macro rules from most-specific to least-specific.

To defend against future syntax changes altering the interpretation of macro input, `macro_rules!`
restricts what can follow various metavariables.
The complete list, showing what may follow what fragment specifier, as of Rust 1.46 is as follows:

* [`stmt`] and [`expr`]: `=>`, `,`, or `;`
* [`pat`]: `=>`, `,`, `=`, `if`, `in`[^pat-edition]
* [`pat_param`]: `=>`, `,`, `=`, `|`, `if`, `in`
* [`path`] and [`ty`]:`=>`, `,`, `=`, `|`, `;`, `:`, `>`, `>>`, `[`, `{`, `as`, `where`, or a macro
    variable of the [`block`] fragment specifier.
* [`vis`]: `,`, an identifier other than a non-raw `priv`, any token that can begin a type or a
    metavariable with an [`ident`], [`ty`], or [`path`] fragment specifier.
* All other fragment specifiers have no restrictions.

[^pat-edition]: **Edition Differences**: Before the 2021 edition, `pat` may also be followed by `|`.

Repetitions also adhere to these restrictions, meaning if a repetition can repeat multiple times(`*` or `+`), then the contents must be able to follow themselves.
If a repetition can repeat zero times (`?` or `*`) then what comes after the repetition must be able to follow what comes before.

The parser also does not perform any kind of lookahead.
That means if the compiler cannot unambiguously determine how to parse the macro invocation one token at a time, it will abort with an ambiguity error.
A simple example that triggers this:

```rust,compile_fail
macro_rules! ambiguity {
    ($($i:ident)* $i2:ident) => { };
}

// error:
//    local ambiguity: multiple parsing options: built-in NTs ident ('i') or ident ('i2').
ambiguity!(an_identifier);
```

The parser does not look ahead past the identifier to see if the following token is a `)`, which would allow it to parse properly.

One aspect of substitution that often surprises people is that substitution is *not* token-based, despite very much *looking* like it.

Consider the following:

```rust
macro_rules! capture_then_match_tokens {
    ($e:expr) => {match_tokens!($e)};
}

macro_rules! match_tokens {
    ($a:tt + $b:tt) => {"got an addition"};
    (($i:ident)) => {"got an identifier"};
    ($($other:tt)*) => {"got something else"};
}

fn main() {
    println!("{}\n{}\n{}\n",
        match_tokens!((caravan)),
        match_tokens!(3 + 6),
        match_tokens!(5));
    println!("{}\n{}\n{}",
        capture_then_match_tokens!((caravan)),
        capture_then_match_tokens!(3 + 6),
        capture_then_match_tokens!(5));
}
```

The output is:

```text
got an identifier
got an addition
got something else

got something else
got something else
got something else
```

By parsing the input into an AST node, the substituted result becomes *un-destructible*;
*i.e.* you cannot examine the contents or match against it ever again.

Here is *another* example which can be particularly confusing:

```rust
macro_rules! capture_then_what_is {
    (#[$m:meta]) => {what_is!(#[$m])};
}

macro_rules! what_is {
    (#[no_mangle]) => {"no_mangle attribute"};
    (#[inline]) => {"inline attribute"};
    ($($tts:tt)*) => {concat!("something else (", stringify!($($tts)*), ")")};
}

fn main() {
    println!(
        "{}\n{}\n{}\n{}",
        what_is!(#[no_mangle]),
        what_is!(#[inline]),
        capture_then_what_is!(#[no_mangle]),
        capture_then_what_is!(#[inline]),
    );
}
```

The output is:

```text
no_mangle attribute
inline attribute
something else (#[no_mangle])
something else (#[inline])
```

The only way to avoid this is to capture using the [`tt`], [`ident`] or [`lifetime`] kinds.
Once you capture with anything else, the only thing you can do with the result from then on is substitute it directly into the output.


[`item`]:./fragment-specifiers.md#item
[`block`]:./fragment-specifiers.md#block
[`stmt`]:./fragment-specifiers.md#stmt
[`pat`]:./fragment-specifiers.md#pat
[`pat_param`]:./fragment-specifiers.md#pat_param
[`expr`]:./fragment-specifiers.md#expr
[`ty`]:./fragment-specifiers.md#ty
[`ident`]:./fragment-specifiers.md#ident
[`path`]:./fragment-specifiers.md#path
[`tt`]:./fragment-specifiers.md#tt
[`meta`]:./fragment-specifiers.md#meta
[`lifetime`]:./fragment-specifiers.md#lifetime
[`vis`]:./fragment-specifiers.md#vis
[`literal`]:./fragment-specifiers.md#literal
