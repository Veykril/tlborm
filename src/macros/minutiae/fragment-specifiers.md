# Fragment Specifiers

As shown in the [`macro_rules`] chapter, Rust, as of 1.46, has 13 fragment specifiers.
This section will go a bit more into detail for some of them and tries to always show a few examples of what a matcher can match with.

> Note that capturing with anything but the `ident`, `lifetime` and `tt` fragments will render the captured AST opaque, making it impossible to further inspect it in future macro invocations.

* [`block`](#block)
* [`expr`](#expr)
* [`ident`](#ident)
* [`item`](#item)
* [`lifetime`](#lifetime)
* [`literal`](#literal)
* [`meta`](#meta)
* [`pat`](#pat)
* [`path`](#path)
* [`stmt`](#stmt)
* [`tt`](#tt)
* [`ty`](#ty)
* [`vis`](#vis)

## `block`

The `block` fragment solely matches a [block expression](https://doc.rust-lang.org/reference/expressions/block-expr.html), which consists of an opening `{` brace, followed by any amount of statements and finally followed by a closing `}` brace.

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

The `expr` fragment matches any kind of [expression](https://doc.rust-lang.org/reference/expressions.html)(Rust has a lot of them, given it *is* an expression orientated language).

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
Item examples:

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

The `meta` fragment matches an [attribute](https://doc.rust-lang.org/reference/attributes.html), to be more precise, the contents of an attribute.
You will usually see this fragment being used in a matcher like `#[$meta:meta]` or `#![$meta:meta]`.

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

> A neat thing about doc comments: They are actually attributes in the form of `#[doc="…"]` where the `...` is the actual comment string, meaning you can act on doc comments in macros!

## `pat`

The `pat` fragment matches any kind of [pattern](https://doc.rust-lang.org/reference/patterns.html).

```rust
macro_rules! patterns {
    ($($pat:pat)*) => ();
}

patterns! {
    "literal"
    _
    0..5
    ref mut PatternsAreNice
}
# fn main() {}
```

## `path`

The `path` fragment matches a so called [TypePath](https://doc.rust-lang.org/reference/paths.html#paths-in-types) style path.

```rust
macro_rules! paths {
    ($($path:path)*) => ();
}

paths! {
    ASimplePath
    ::A::B::C::D
    G::<eneri>::C
}
# fn main() {}
```

## `stmt`

The `statement` fragment solely matches a [statement](https://doc.rust-lang.org/reference/statements.html) without its trailing semicolon, unless its an item statement that requires one.
What would be an item statement that requires one?
A Unit-Struct would be a simple one, as defining one requires a trailing semicolon.

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

From this we can tell a few things:

The first you should be able to see immediately is that while the `stmt` fragment doesn't capture trailing semicolons, it still emits them when required, even if the statement is already followed by one.
The simple reason for that is that semicolons on their own are already valid statements.
So we are actually invoking our macro here with not 8 statements, but 11!

Another thing you should be able to notice here is that the trailing semicolon of the `struct Foo;` item statement is being matched, otherwise we would've seen an extra one like in the other cases.
This makes sense as we already said, that for item statements that require one, the trailing semicolon will be matched with.

A last observation is that expressions get emitted back with a trailing semicolon, unless the expression solely consists of only a block expression or control flow expression.

The fine details of what was just mentioned here can be looked up in the [reference](https://doc.rust-lang.org/reference/statements.html).

[^debugging]:See the [debugging chapter](./debugging.md) for tips on how to do this.

## `tt`

The `tt` fragment matches a TokenTree.
If you need a refresher on what exactly a TokenTree was you may want to revisit the [TokenTree chapter](../../syntax-extensions/source-analysis.md#token-trees) of this book.
The `tt` fragment is one of the most powerful fragments, as it can match nearly anything while still allowing you to inspect the contents of it at a later state in the macro.

## `ty`

The `ty` fragment matches any kind of [type expression](https://doc.rust-lang.org/reference/types.html#type-expressions).
A type expression is the syntax with which one refers to a type in the language.

```rust
macro_rules! types {
    ($($type:ty)*) => ();
}

types! {
    foo::bar
    bool
    [u8]
}
# fn main() {}
```

## `vis`

The `vis` fragment matches a *possibly empty* [Visibility qualifier](https://doc.rust-lang.org/reference/visibility-and-privacy.html).
Emphasis lies on the *possibly empty* part.
You can think of this fragment having an implicit `?` repetition to it, meaning you don't, and in fact cannot, wrap it in a direct repetition.

```rust
macro_rules! visibilities {
    //         ∨~~Note this comma, since we cannot repeat a `vis` fragment on its own
    ($($vis:vis,)*) => ();
}

visibilities! {
    ,
    pub,
    pub(crate),
    pub(in super),
    pub(in some_path),
}
# fn main() {}
```

[`macro_rules`]: ../macro_rules.md
