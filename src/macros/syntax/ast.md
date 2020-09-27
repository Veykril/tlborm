# Macros in the AST

As previously mentioned, macro processing in Rust happens *after* the construction of the AST. As
such, the syntax used to invoke a macro *must* be a proper part of the language's syntax. In fact,
there are several "syntax extension" forms which are part of Rust's syntax. Specifically, the
following forms (by way of examples):

* `# [ $arg ]`; *e.g.* `#[derive(Clone)]`, `#[no_mangle]`, …
* `# ! [ $arg ]`; *e.g.* `#![allow(dead_code)]`, `#![crate_name="blang"]`, …
* `$name ! $arg`; *e.g.* `println!("Hi!")`, `concat!("a", "b")`, …
* `$name ! $arg0 $arg1`; *e.g.* `macro_rules! dummy { () => {}; }`.

The first two are [attributes] which annotate items, expressions and statements. They can be
classified into different kinds, [built-in attributes], [macro attributes] and [derive attributes].
The latter two can be implemented with the second macro system that Rust offers, [procedural macros].

The third is the one of interest to us: function-like macros. It is the form available for use with
`macro_rules!` macros. Note that this form is not *limited* to `macro_rules!` macros: it is a
generic syntax extension form. For example, whilst [`format!`] is a macro, [`format_args!`] (which
is used to *implement* [`format!`]) is *not*.

The fourth is essentially a variation which is *not* available to macros. In fact, the only case
where this form is used *at all* is with `macro_rules!` itself which, again we will come back to.

Disregarding all but the third form (`$name ! $arg`), the question becomes: how does the Rust parser
know what `$arg` looks like for every possible syntax extension? The answer is that it doesn't
*have to*. Instead, the argument of a syntax extension invocation is a *single* token tree. More
specifically, it is a single, *non-leaf* token tree; `(...)`, `[...]`, or `{...}`. With that
knowledge, it should become apparent how the parser can understand all of the following invocation
forms:

```rust,ignore
bitflags! {
    struct Color: u8 {
        const RED    = 0b0001,
        const GREEN  = 0b0010,
        const BLUE   = 0b0100,
        const BRIGHT = 0b1000,
    }
}

lazy_static! {
    static ref FIB_100: u32 = {
        fn fib(a: u32) -> u32 {
            match a {
                0 => 0,
                1 => 1,
                a => fib(a-1) + fib(a-2)
            }
        }

        fib(100)
    };
}

fn main() {
    use Color::*;
    let colors = vec![RED, GREEN, BLUE];
    println!("Hello, World!");
}
```

Although the above invocations may *look* like they contain various kinds of Rust code, the parser
simply sees a collection of meaningless token trees. To make this clearer, we can replace all these
syntactic "black boxes" with ⬚, leaving us with:

```text
bitflags! ⬚

lazy_static! ⬚

fn main() {
    let colors = vec! ⬚;
    println! ⬚;
}
```

Just to reiterate: the parser does not assume *anything* about ⬚; it remembers the tokens it
contains, but doesn't try to *understand* them. This means ⬚ can be anything, even invalid Rust! As
to why this is a good thing, we will come back to that at a later point.

The important takeaways are:

* There are multiple kinds of syntax extension in Rust. We will *only* be talking about macros
    defined by the `macro_rules!` construct.
* Just because you see something of the form `$name! $arg`, doesn't mean it's actually a macro; it
    might be another kind of syntax extension, a procedural macro for example.
* The input to every macro is a single non-leaf token tree.
* Macros (really, syntax extensions in general) are parsed as *part* of the abstract syntax tree.

> **Aside**: due to the first point, some of what will be said below (including the next paragraph)
> will apply to syntax extensions *in general*.[^writer-is-lazy]

[^writer-is-lazy]: This is rather convenient as "macro" is much quicker and easier to type than
"syntax extension".

The last point is the most important, as it has *significant* implications. Because macros are
parsed into the AST, they can **only** appear in positions where they are explicitly supported.
Specifically macros can appear in place of the following:

* Patterns
* Statements
* Expressions
* Items(this includes `impl` Items)
* Types

Some things *not* on this list:

* Identifiers
* Match arms
* Struct fields

There is absolutely, definitely *no way* to use macros in any position *not* on the first list.

[attributes]: https://doc.rust-lang.org/reference/attributes.html
[built-in attributes]: https://doc.rust-lang.org/reference/attributes.html#built-in-attributes-index
[macro attributes]: https://doc.rust-lang.org/reference/procedural-macros.html#attribute-macros
[derive attributes]: https://doc.rust-lang.org/reference/procedural-macros.html#derive-macro-helper-attributes
[procedural macros]: https://doc.rust-lang.org/reference/procedural-macros.html
[`format!`]: https://doc.rust-lang.org/std/macro.format.html
[`format_args!`]: https://doc.rust-lang.org/std/macro.format_args.html
