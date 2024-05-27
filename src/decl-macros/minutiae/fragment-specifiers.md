# Fragment Specifiers

As mentioned in the [`methodical introduction`](../macros-methodical.md) chapter, Rust, as of 1.60, has 14 fragment specifiers.
This section will go a bit more into detail for some of them and shows a few example inputs of what each matcher matches.

> **Note**: Capturing with anything but the `ident`, `lifetime` and `tt` fragments will render the captured AST opaque, making it impossible to further match it with other fragment specifiers in future macro invocations.

* [`block`](#block)
* [`expr`](#expr)
* [`ident`](#ident)
* [`item`](#item)
* [`lifetime`](#lifetime)
* [`literal`](#literal)
* [`meta`](#meta)
* [`pat`](#pat)
* [`pat_param`](#pat_param)
* [`path`](#path)
* [`stmt`](#stmt)
* [`tt`](#tt)
* [`ty`](#ty)
* [`vis`](#vis)

## `block`

The `block` fragment solely matches a [block expression](https://doc.rust-lang.org/reference/expressions/block-expr.html), which consists of an opening `{` brace, followed by any number of statements and finally followed by a closing `}` brace.

```rust
macro_rules! blocks {
    ($($block:block)*) => ();
}

blocks! {
    {}
    {
        let zig;
    }
    { 2 }
}
# fn main() {}
```

## `expr`

The `expr` fragment matches any kind of [expression](https://doc.rust-lang.org/reference/expressions.html) (Rust has a lot of them, given it *is* an expression-oriented language).

```rust
macro_rules! expressions {
    ($($expr:expr)*) => ();
}

expressions! {
    "literal"
    funcall()
    future.await
    break 'foo bar
}
# fn main() {}
```

## `ident`

The `ident` fragment matches an [identifier](https://doc.rust-lang.org/reference/identifiers.html) or *keyword*.

```rust
macro_rules! idents {
    ($($ident:ident)*) => ();
}

idents! {
    // _ <- This is not an ident, it is a pattern
    foo
    async
    O_________O
    _____O_____
}
# fn main() {}
```

## `item`

The `item` fragment simply matches any of Rust's [item](https://doc.rust-lang.org/reference/items.html) *definitions*, not identifiers that refer to items.
This includes visibility modifiers.

```rust
macro_rules! items {
    ($($item:item)*) => ();
}

items! {
    struct Foo;
    enum Bar {
        Baz
    }
    impl Foo {}
    pub use crate::foo;
    /*...*/
}
# fn main() {}
```

## `lifetime`

The `lifetime` fragment matches a [lifetime or label](https://doc.rust-lang.org/reference/tokens.html#lifetimes-and-loop-labels).
It's quite similar to [`ident`](#ident) but with a prepended `'`.

```rust
macro_rules! lifetimes {
    ($($lifetime:lifetime)*) => ();
}

lifetimes! {
    'static
    'shiv
    '_
}
# fn main() {}
```

## `literal`

The `literal` fragment matches any [literal expression](https://doc.rust-lang.org/reference/expressions/literal-expr.html).

```rust
macro_rules! literals {
    ($($literal:literal)*) => ();
}

literals! {
    -1
    "hello world"
    2.3
    b'b'
    true
}
# fn main() {}
```

## `meta`

The `meta` fragment matches the contents of an [attribute](https://doc.rust-lang.org/reference/attributes.html).
That is, it will match a simple path, one without generic arguments followed by a delimited token tree or an `=` followed by a literal expression.

> **Note**: You will usually see this fragment being used in a matcher like `#[$meta:meta]` or `#![$meta:meta]` to actually capture an attribute.

```rust
macro_rules! metas {
    ($($meta:meta)*) => ();
}

metas! {
    ASimplePath
    super::man
    path = "home"
    foo(bar)
}
# fn main() {}
```

> **Doc-Comment Fact**: Doc-Comments like `/// ...` and `//! ...` are actually syntax sugar for attributes! They desugar to `#[doc="..."]` and `#![doc="..."]` respectively, meaning you can match on them like with attributes!

## `pat`

The `pat` fragment matches any kind of [pattern](https://doc.rust-lang.org/reference/patterns.html), including or-patterns starting with the 2021 edition.

```rust,edition2021
macro_rules! patterns {
    ($($pat:pat)*) => ();
}

patterns! {
    "literal"
    _
    0..5
    ref mut PatternsAreNice
    0 | 1 | 2 | 3
}
# fn main() {}
```

## `pat_param`

In the 2021 edition, the behavior for the `pat` fragment type has been changed to allow or-patterns to be parsed.
This changes the follow list of the fragment, preventing such fragment from being followed by a `|` token.
To avoid this problem or to get the old fragment behavior back one can use the `pat_param` fragment which allows `|` to follow it, as it disallows top level or-patterns.

```rust
macro_rules! patterns {
    ($( $( $pat:pat_param )|+ )*) => ();
}

patterns! {
    "literal"
    _
    0..5
    ref mut PatternsAreNice
    0 | 1 | 2 | 3
}
# fn main() {}
```

## `path`

The `path` fragment matches a so called [TypePath](https://doc.rust-lang.org/reference/paths.html#paths-in-types) style path.
This includes the function style trait forms, `Fn() -> ()`.

```rust
macro_rules! paths {
    ($($path:path)*) => ();
}

paths! {
    ASimplePath
    ::A::B::C::D
    G::<eneri>::C
    FnMut(u32) -> ()
}
# fn main() {}
```

## `stmt`

The `statement` fragment solely matches a [statement](https://doc.rust-lang.org/reference/statements.html) without its trailing semicolon, unless it is an item statement that requires one (such as a Unit-Struct).

Let's use a simple example to show exactly what is meant with this.
We use a macro that merely emits what it captures:

```rust,ignore
macro_rules! statements {
    ($($stmt:stmt)*) => ($($stmt)*);
}

fn main() {
    statements! {
        struct Foo;
        fn foo() {}
        let zig = 3
        let zig = 3;
        3
        3;
        if true {} else {}
        {}
    }
}

```

Expanding this, via the [playground](https://play.rust-lang.org/) for example[^debugging], gives us roughly the following:

```rust,ignore
/* snip */

fn main() {
    struct Foo;
    fn foo() { }
    let zig = 3;
    let zig = 3;
    ;
    3;
    3;
    ;
    if true { } else { }
    { }
}
```

From this we can tell a few things.

The first you should be able to see immediately is that while the `stmt` fragment doesn't capture trailing semicolons, it still emits them when required, even if the statement is already followed by one.
The simple reason for that is that semicolons on their own are already valid statements which the fragment captures eagerly.
So our macro isn't capturing 8 times, but 10!
This can be important when doing multiples repetitions and expanding these in one repetition expansion, as the repetition numbers have to match in those cases.

Another thing you should be able to notice here is that the trailing semicolon of the `struct Foo;` item statement is being matched, otherwise we would've seen an extra one like in the other cases.
This makes sense as we already said, that for item statements that require one, the trailing semicolon will be matched with.

A last observation is that expressions get emitted back with a trailing semicolon, unless the expression solely consists of only a block expression or control flow expression.

The fine details of what was just mentioned here can be looked up in the [reference](https://doc.rust-lang.org/reference/statements.html).

Fortunately, these fine details here are usually not of importance whatsoever, with the small exception that was mentioned earlier in regards to repetitions which by itself shouldn't be a common problem to run into.

[^debugging]:See the [debugging chapter](./debugging.md) for tips on how to do this.

## `tt`

The `tt` fragment matches a TokenTree.
If you need a refresher on what exactly a TokenTree was you may want to revisit the [TokenTree chapter](../../syntax-extensions/source-analysis.md#token-trees) of this book.
The `tt` fragment is one of the most powerful fragments, as it can match nearly anything while still allowing you to inspect the contents of it at a later state in the macro.

This allows one to make use of very powerful patterns like the [tt-muncher](../patterns/tt-muncher.md) or the [push-down-accumulator](../patterns/push-down-acc.md).

## `ty`

The `ty` fragment matches any kind of [type expression](https://doc.rust-lang.org/reference/types.html#type-expressions).

```rust
macro_rules! types {
    ($($type:ty)*) => ();
}

types! {
    foo::bar
    bool
    [u8]
    impl IntoIterator<Item = u32>
}
# fn main() {}
```

## `vis`

The `vis` fragment matches a *possibly empty* [Visibility qualifier].

```rust
macro_rules! visibilities {
    //         ∨~~Note this comma, since we cannot repeat a `vis` fragment on its own
    ($($vis:vis,)*) => ();
}

visibilities! {
    , // no vis is fine, due to the implicit `?`
    pub,
    pub(crate),
    pub(in super),
    pub(in some_path),
}
```

While able to match empty sequences of tokens, the fragment specifier still acts quite different from [optional repetitions](../macros-methodical.md#repetitions) which is described in the following:

If it is being matched against no left over tokens the entire macro matching fails.
```rust,compile_fail
macro_rules! non_optional_vis {
    ($vis:vis) => ();
}
non_optional_vis!();
// ^^^^^^^^^^^^^^^^ error: missing tokens in macro arguments
```

`$vis:vis $ident:ident` matches fine.
```rust
macro_rules! vis_ident {
    ($vis:vis $ident:ident) => ();
}
vis_ident!(pub foo); // this works fine
```

In contrast, `$(pub)? $ident:ident` is ambiguous, as `pub` denotes a valid identifier.
```rust,compile_fail
macro_rules! pub_ident {
    ($(pub)? $ident:ident) => ();
}
pub_ident!(pub foo);
        // ^^^ error: local ambiguity when calling macro `pub_ident`: multiple parsing options: built-in NTs ident ('ident') or 1 other option.
```

Being a fragment that matches the empty token sequence also gives it a very interesting quirk in combination with `tt` fragments and recursive expansions.

When matching the empty token sequence, the metavariable will still count as a capture and since it is not a `tt`, `ident` or `lifetime` fragment it will become opaque to further expansions.
This means if this capture is passed onto another macro invocation that captures it as a `tt` you effectively end up with token tree that contains nothing!

```rust
macro_rules! it_is_opaque {
    (()) => { "()" };
    (($tt:tt)) => { concat!("$tt is ", stringify!($tt)) };
    ($vis:vis ,) => { it_is_opaque!( ($vis) ); }
}
fn main() {
    // this prints "$tt is ", as the recursive calls hits the second branch with
    // an empty tt, opposed to matching with the first branch!
    println!("{}", it_is_opaque!(,));
}
```

[`macro_rules`]: ../macro_rules.md
[Visibility qualifier]: https://doc.rust-lang.org/reference/visibility-and-privacy.html
