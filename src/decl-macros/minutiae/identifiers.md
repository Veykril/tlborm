# Non-Identifier Identifiers

There are two tokens which you are likely to run into eventually that *look* like identifiers, but aren't.
Except when they are.

First is `self`.
This is *very definitely* a keyword.
However, it also happens to fit the definition of an identifier.
In regular Rust code, there's no way for `self` to be interpreted as an identifier, but it *can* happen with `macro_rules!` macros:

```rust
macro_rules! what_is {
    (self) => {"the keyword `self`"};
    ($i:ident) => {concat!("the identifier `", stringify!($i), "`")};
}

macro_rules! call_with_ident {
    ($c:ident($i:ident)) => {$c!($i)};
}

fn main() {
    println!("{}", what_is!(self));
    println!("{}", call_with_ident!(what_is(self)));
}
```

The above outputs:

```text
the keyword `self`
the keyword `self`
```

But that makes no sense; `call_with_ident!` required an identifier, matched one, and substituted it!
So `self` is both a keyword and not a keyword at the same time.
You might wonder how this is in any way important.
Take this example:

```rust,compile_fail
macro_rules! make_mutable {
    ($i:ident) => {let mut $i = $i;};
}

struct Dummy(i32);

impl Dummy {
    fn double(self) -> Dummy {
        make_mutable!(self);
        self.0 *= 2;
        self
    }
}
#
# fn main() {
#     println!("{:?}", Dummy(4).double().0);
# }
```

This fails to compile with:

```text
error: `mut` must be followed by a named binding
 --> src/main.rs:2:24
  |
2 |     ($i:ident) => {let mut $i = $i;};
  |                        ^^^^^^ help: remove the `mut` prefix: `self`
...
9 |         make_mutable!(self);
  |         -------------------- in this macro invocation
  |
  = note: `mut` may be followed by `variable` and `variable @ pattern`
```

So the macro will happily match `self` as an identifier, allowing you to use it in cases where you can't actually use it.
But, fine; it somehow remembers that `self` is a keyword even when it's an identifier, so you *should* be able to do this, right?

```rust,compile_fail
macro_rules! make_self_mutable {
    ($i:ident) => {let mut $i = self;};
}

struct Dummy(i32);

impl Dummy {
    fn double(self) -> Dummy {
        make_self_mutable!(mut_self);
        mut_self.0 *= 2;
        mut_self
    }
}
#
# fn main() {
#     println!("{:?}", Dummy(4).double().0);
# }
```

This fails with:

```text
error[E0424]: expected value, found module `self`
  --> src/main.rs:2:33
   |
2  |       ($i:ident) => {let mut $i = self;};
   |                                   ^^^^ `self` value is a keyword only available in methods with a `self` parameter
...
8  | /     fn double(self) -> Dummy {
9  | |         make_self_mutable!(mut_self);
   | |         ----------------------------- in this macro invocation
10 | |         mut_self.0 *= 2;
11 | |         mut_self
12 | |     }
   | |_____- this function has a `self` parameter, but a macro invocation can only access identifiers it receives from parameters
   |
```

Now the compiler thinks we refer to our module with `self`, but that doesn't make sense.
We already have a `self` right there, in the function signature which is definitely not a module.
It's almost like it's complaining that the `self` it's trying to use isn't the *same* `self`... as though the `self` keyword has hygiene, like an... identifier.

```rust,compile_fail
macro_rules! double_method {
    ($body:expr) => {
        fn double(mut self) -> Dummy {
            $body
        }
    };
}

struct Dummy(i32);

impl Dummy {
    double_method! {{
        self.0 *= 2;
        self
    }}
}
#
# fn main() {
#     println!("{:?}", Dummy(4).double().0);
# }
```

Same error.  What about...

```rust
macro_rules! double_method {
    ($self_:ident, $body:expr) => {
        fn double(mut $self_) -> Dummy {
            $body
        }
    };
}

struct Dummy(i32);

impl Dummy {
    double_method! {self, {
        self.0 *= 2;
        self
    }}
}
#
# fn main() {
#     println!("{:?}", Dummy(4).double().0);
# }
```

At last, *this works*.
So `self` is both a keyword *and* an identifier when it feels like it.
Surely this works for other, similar constructs, right?

```rust,compile_fail
macro_rules! double_method {
    ($self_:ident, $body:expr) => {
        fn double($self_) -> Dummy {
            $body
        }
    };
}

struct Dummy(i32);

impl Dummy {
    double_method! {_, 0}
}
#
# fn main() {
#     println!("{:?}", Dummy(4).double().0);
# }
```

```text
error: no rules expected the token `_`
  --> src/main.rs:12:21
   |
1  | macro_rules! double_method {
   | -------------------------- when calling this macro
...
12 |     double_method! {_, 0}
   |                     ^ no rules expected this token in macro call
```

No, of course not.
`_` is a keyword that is valid in patterns and expressions, but somehow *isn't* an identifier like the keyword `self` is, despite matching the definition of an identifier just the same.

You might think you can get around this by using `$self_:pat` instead; that way, `_` will match! Except, no, because `self` isn't a pattern.
Joy.

The only work around for this (in cases where you want to accept some combination of these tokens) is to use a [`tt`] matcher instead.

[`tt`]:./fragment-specifiers.md#tt
