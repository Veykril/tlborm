# Expansion

Expansion is a relatively simple affair.
At some point *after* the construction of the AST, but before the compiler begins constructing its semantic understanding of the program, it will expand all syntax extensions.

This involves traversing the AST, locating syntax extension invocations and replacing them with their expansion.

Once the compiler has run a syntax extension, it expects the result to be parsable as one of a limited set of syntax elements, based on context.
For example, if you invoke a syntax extension at module scope, the compiler will parse the result into an AST node that represents an item.
If you invoke a syntax extension in expression position, the compiler will parse the result into an expression AST node.

In fact, it can turn a syntax extension result into any of the following:

* an expression,
* a pattern,
* a type,
* zero or more items, or
* zero or more statements.

In other words, *where* you can invoke a syntax extension determines what its result will be interpreted as.

The compiler will take this AST node and completely replace the syntax extension's invocation node with the output node.
*This is a structural operation*, not a textual one!

For example, consider the following:

```rust,ignore
let eight = 2 * four!();
```

We can visualize this partial AST as follows:

```text
┌─────────────┐
│ Let         │
│ name: eight │   ┌─────────┐
│ init: ◌     │╶─╴│ BinOp   │
└─────────────┘   │ op: Mul │
                ┌╴│ lhs: ◌  │
     ┌────────┐ │ │ rhs: ◌  │╶┐ ┌────────────┐
     │ LitInt │╶┘ └─────────┘ └╴│ Macro      │
     │ val: 2 │                 │ name: four │
     └────────┘                 │ body: ()   │
                                └────────────┘
```

From context, `four!()` *must* expand to an expression (the initializer can *only* be an expression).
Thus, whatever the actual expansion is, it will be interpreted as a complete expression.
In this case, we will assume `four!` is defined such that it expands to the expression `1 + 3`.
As a result, expanding this invocation will result in the AST changing to:

```text
┌─────────────┐
│ Let         │
│ name: eight │   ┌─────────┐
│ init: ◌     │╶─╴│ BinOp   │
└─────────────┘   │ op: Mul │
                ┌╴│ lhs: ◌  │
     ┌────────┐ │ │ rhs: ◌  │╶┐ ┌─────────┐
     │ LitInt │╶┘ └─────────┘ └╴│ BinOp   │
     │ val: 2 │                 │ op: Add │
     └────────┘               ┌╴│ lhs: ◌  │
                   ┌────────┐ │ │ rhs: ◌  │╶┐ ┌────────┐
                   │ LitInt │╶┘ └─────────┘ └╴│ LitInt │
                   │ val: 1 │                 │ val: 3 │
                   └────────┘                 └────────┘
```

This can be written out like so:

```rust,ignore
let eight = 2 * (1 + 3);
```

Note that we added parentheses *despite* them not being in the expansion.
Remember that the compiler always treats the expansion of a syntax extension as a complete AST node, **not** as a mere sequence of tokens.
To put it another way, even if you don't explicitly wrap a complex expression in parentheses, there is no way for the compiler to "misinterpret" the result, or change the order of evaluation.

It is important to understand that syntax extension expansions are treated as AST nodes, as this design has two further implications:

* In addition to there being a limited number of invocation *positions*, syntax extension can *only* expand to the kind of AST node the parser *expects* at that position.
* As a consequence of the above, syntax extension  *absolutely cannot* expand to incomplete or syntactically invalid constructs.

There is one further thing to note about expansion: what happens when a syntax extension expands to something that contains *another* syntax extension invocation.
For example, consider an alternative definition of `four!`; what happens if it expands to `1 + three!()`?

```rust,ignore
let x = four!();
```

Expands to:

```rust,ignore
let x = 1 + three!();
```

This is resolved by the compiler checking the result of expansions for additional syntax extension invocations, and expanding them.
Thus, a second expansion step turns the above into:

```rust,ignore
let x = 1 + 3;
```

The takeaway here is that expansion happens in "passes";
as many as is needed to completely expand all invocations.

Well, not *quite*.
In fact, the compiler imposes an upper limit on the number of such recursive passes it is willing to run before giving up.
This is known as the syntax extension  recursion limit and defaults to 128.
If the 128nd expansion contains a syntax extension invocation, the compiler will abort with an error indicating that the recursion limit was exceeded.

This limit can be raised using the `#![recursion_limit="…"]` [attribute][recursion_limit], though it *must* be done crate-wide.
Generally, it is recommended to try and keep syntax extension below this limit wherever possible as it may impact compilation times.

[recursion_limit]: https://doc.rust-lang.org/reference/attributes/limits.html#the-recursion_limit-attribute
