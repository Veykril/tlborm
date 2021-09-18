# Macros in the AST

As previously mentioned, macro processing in Rust happens *after* the construction of the AST.
As such, the syntax used to invoke a macro *must* be a proper part of the language's syntax.
In fact, there are several "syntax extension" forms which are part of Rust's syntax.
Specifically, the following 4 forms (by way of examples):

1. `# [ $arg ]`; *e.g.* `#[derive(Clone)]`, `#[no_mangle]`, …
2. `# ! [ $arg ]`; *e.g.* `#![allow(dead_code)]`, `#![crate_name="blang"]`, …
3. `$name ! $arg`; *e.g.* `println!("Hi!")`, `concat!("a", "b")`, …
4. `$name ! $arg0 $arg1`; *e.g.* `macro_rules! dummy { () => {}; }`.

The first two are [attributes] which annotate items, expressions and statements. They can be
classified into different kinds, [built-in attributes], [proc-macro attributes] and [derive attributes].
[proc-macro attributes] and [derive attributes] can be implemented with the second macro system that Rust
offers, [procedural macros]. [built-in attributes] on the other hand are attributes implemented by
the compiler.

The third form `$name ! $arg` are function-like macros. It is the form available for use with `macro_rules!`, `macro` and also procedural macros.
Note that this form is not *limited* to `macro_rules!` macros: it is a generic syntax extension form.
For example, whilst [`format!`] is a `macro_rules!` macro, [`format_args!`] (which is used to *implement* [`format!`]) is *not* as it is a compiler builtin.


The fourth form is essentially a variation which is *not* available to macros.
In fact, the only case where this form is used *at all* is with the `macro_rules!` construct itself.

So, starting with the third form, how does the Rust parser know what the `$arg` in (`$name ! $arg`) looks like for every possible syntax extension?
The answer is that it doesn't *have to*.
Instead, the argument of a syntax extension invocation is a *single* token tree.
More specifically, it is a single, *non-leaf* token tree; `(...)`, `[...]`, or `{...}`. With that
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

Although the above invocations may *look* like they contain various kinds of Rust code, the parser simply sees a collection of meaningless token trees.
To make this clearer, we can replace all these syntactic "black boxes" with ⬚, leaving us with:

```text
bitflags! ⬚

lazy_static! ⬚

fn main() {
    let colors = vec! ⬚;
    println! ⬚;
}
```

Just to reiterate: the parser does not assume *anything* about ⬚;
it remembers the tokens it contains, but doesn't try to *understand* them.
This means ⬚ can be anything, even invalid Rust!
As to why this is a good thing, we will come back to that at a later point.

So, does this also apply to `$arg` in form 1 and 2, and to ther two args in form 4? Kind of.
The `$arg$` for form 1 and 2 is a bit different in that it is not directly a token tree, but a *simple path* that is either followed by an `=` token and a literal expression, or a token tree.
We will explore this more in-depth in the appropriate proc-macro chapter.
The important part here is that this form as well, makes use of token trees to describe the input.
The 4th form in general is more special and accepts a very specific grammar that also makes use of token trees though.
The specifics of this form do not matter at this point so we will skip them until they become relevant.

The important takeaways from this are:

* There are multiple kinds of syntax extensions in Rust.
* Just seeing something of the form `$name! $arg`, doesn't tell you what kind of syntax extension it might be.
    It could be a `macro_rules!` macro, a `proc-macro` or maybe even a builtin.
* The input to every `!` macro invocation, that is form 3, is a single non-leaf token tree.
* Syntax extensions are parsed as *part* of the abstract syntax tree.

The last point is the most important, as it has *significant* implications.
Because syntax extensions are parsed into the AST, they can **only** appear in positions where they are explicitly supported.
Specifically syntax extensions can appear in place of the following:

* Patterns
* Statements
* Expressions
* Items(this includes `impl` items)
* Types

Some things *not* on this list:

* Identifiers
* Match arms
* Struct fields

There is absolutely, definitely *no way* to use syntax extensions in any position *not* on the first list.

[attributes]: https://doc.rust-lang.org/reference/attributes.html
[built-in attributes]: https://doc.rust-lang.org/reference/attributes.html#built-in-attributes-index
[proc-macro attributes]: https://doc.rust-lang.org/reference/procedural-macros.html#attribute-macros
[derive attributes]: https://doc.rust-lang.org/reference/procedural-macros.html#derive-macro-helper-attributes
[procedural macros]: https://doc.rust-lang.org/reference/procedural-macros.html
[`format!`]: https://doc.rust-lang.org/std/macro.format.html
[`format_args!`]: https://doc.rust-lang.org/std/macro.format_args.html
