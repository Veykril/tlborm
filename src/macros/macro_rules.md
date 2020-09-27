# `macro_rules!`

With all that in mind, we can introduce `macro_rules!` itself. As noted previously, `macro_rules!`
is *itself* a syntax extension, meaning it is *technically* not part of the Rust syntax. It uses the
following forms:

```rust,ignore
macro_rules! $name {
    $rule0 ;
    $rule1 ;
    // …
    $ruleN ;
}
```

There must be *at least* one rule, and you can omit the semicolon after the last rule. You can use
brackets(`[]`), parentheses(`()`) or braces(`{}`). Invocations with `{ .. }` and `( ... );`, notice
the trailing semicolon, will *always* be parsed as an *item*.

Each *"rule"* looks like the following:

```ignore
    ($matcher) => {$expansion}
```

Like before, the types of parens used can be any kind, but parens around the matcher and braces
around the expansion are somewhat conventional. The expansion part of a rule is also called its
*transcriber*.

If you are wondering, the `macro_rules!` invocation expands to... *nothing*.  At least, nothing that
appears in the AST; rather, it manipulates compiler-internal structures to register the macro. As
such, you can *technically* use `macro_rules!` in any position where an empty expansion is valid.

## Matching

When a `macro_rules!` macro is invoked, the `macro_rules!` interpreter goes through the rules one by
one, in declaration order. For each rule, it tries to match the contents of the input token tree
against that rule's `matcher`. A matcher must match the *entirety* of the input to be considered a
match.

If the input matches the matcher, the invocation is replaced by the `expansion`; otherwise, the next
rule is tried. If all rules fail to match, macro expansion fails with an error.

The simplest example is of an empty matcher:

```rust,ignore
macro_rules! four {
    () => { 1 + 3 };
}
```

This matches if and only if the input is also empty (*i.e.* `four!()`, `four![]` or `four!{}`).

Note that the specific grouping tokens you use when you invoke the macro *are not* matched. That is,
you can invoke the above macro as `four![]` and it will still match. Only the *contents* of the
input token tree are considered.

Matchers can also contain literal token trees, which must be matched exactly. This is done by simply
writing the token trees normally. For example, to match the sequence `4 fn ['spang "whammo"] @_@`,
you would write:

```rust,ignore
macro_rules! gibberish {
    (4 fn ['spang "whammo"] @_@) => {...};
}
```

You can use any token tree that you can write.

## Metavariables

Matchers can also contain captures. These allow input to be matched based on some general grammar
category, with the result captured to a metavariable which can then be substituted into the output.

Captures are written as a dollar (`$`) followed by an identifier, a colon (`:`), and finally the
kind of capture which is also called the fragment-specifier, which must be one of the following:

* `block`: a block (i.e. a block of statements and/or an expression, surrounded by braces)
* `expr`: an expression
* `ident`: an identifier (this includes keywords)
* `item`: an item, like a function, struct, module, impl, etc.
* `lifetime`: a lifetime (e.g. `'foo`, `'static`, ...)
* `literal`: a literal (e.g. `"Hello World!"`, `3.14`, `'🦀'`, ...)
* `meta`: a meta item; the things that go inside the `#[...]` and `#![...]` attributes
* `pat`: a pattern
* `path`: a path (e.g. `foo`, `::std::mem::replace`, `transmute::<_, int>`, …)
* `stmt`: a statement
* `tt`: a single token tree
* `ty`: a type
* `vis`: a possible empty visibility qualifier (e.g. `pub`, `pub(in crate)`, ...)

For more in-depth description of the fragement specifiers, check out the
[Fragment Specifiers](./minutiae/fragment-specifiers.md) chapter.

For example, here is a `macro_rules!` macro which captures its input as an expression under the
metavariable `$e`:

```rust,ignore
macro_rules! one_expression {
    ($e:expr) => {...};
}
```

These metavariables leverage the Rust compiler's parser, ensuring that they are always "correct". An
`expr` metavariables will *always* capture a complete, valid expression for the version of Rust being
compiled.

You can mix literal token trees and metavariables, within limits ([explained below]).

To refer to a metavariable you simply write `$name`, as the type of the variable is already
specified in the matcher. For example:

```rust,ignore
macro_rules! times_five {
    ($e:expr) => { 5 * $e };
}
```

Much like macro expansion, metavariables are substituted as complete AST nodes. This means that no
matter what sequence of tokens is captured by `$e`, it will be interpreted as a single, complete
expression.

You can also have multiple metavariables in a single matcher:

```rust,ignore
macro_rules! multiply_add {
    ($a:expr, $b:expr, $c:expr) => { $a * ($b + $c) };
}
```

And use them as often as you like in the expansion:

```rust,ignore
macro_rules! discard {
    ($e:expr) => {};
}
macro_rules! repeat {
    ($e:expr) => { $e; $e; $e; };
}
```

There is also a special metavariable called [`$crate`] which can be used to refer to the current
current.

[`$crate`]:./minutiae/hygiene.html#crate

## Repetitions

Matchers can contain repetitions. These allow a sequence of tokens to be matched. These have the
general form `$ ( ... ) sep rep`.

* `$` is a literal dollar token.
* `( ... )` is the paren-grouped matcher being repeated.
* `sep` is an *optional* separator token. It may not be a delimiter or one
    of the repetition operators. Common examples are `,` and `;`.
* `rep` is the *required* repeat operator. Currently, this can be:
    * `?`: indicating at most one repetition
    * `*`: indicating zero or more repetitions
    * `+`: indicating one or more repetitions

    Since `?` represents at most one occurrence, it cannot be used with a separator.

Repetitions can contain any other valid matcher, including literal token trees, metavariables, and other
repetitions allowing arbitrary nesting.

Repetitions use the same syntax in the expansion and repeated metavariables can only be accessed
inside of repetitions in the expansion.

For example, below is a macro which formats each element as a string. It matches zero or more
comma-separated expressions and expands to an expression that constructs a vector.

```rust
macro_rules! vec_strs {
    (
        // Start a repetition:
        $(
            // Each repeat must contain an expression...
            $element:expr
        )
        // ...separated by commas...
        ,
        // ...zero or more times.
        *
    ) => {
        // Enclose the expansion in a block so that we can use
        // multiple statements.
        {
            let mut v = Vec::new();

            // Start a repetition:
            $(
                // Each repeat will contain the following statement, with
                // $element replaced with the corresponding expression.
                v.push(format!("{}", $element));
            )*

            v
        }
    };
}

fn main() {
    let s = vec_strs![1, "a", true, 3.14159f32];
    assert_eq!(s, &["1", "a", "true", "3.14159"]);
}
```

For the compelte grammar definition you may want to consult the 
[Macros By Example](https://doc.rust-lang.org/reference/macros-by-example.html#macros-by-example)
chapter of the Rust reference.