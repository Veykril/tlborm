# Parsing Rust

Parsing some of Rust's items can be useful in certain situations.
This section will show a few macros that can parse some of Rust's more complex items like structs and functions to a certain extent.
The goal of these macros is not to be able to parse the entire grammar of the items but to parse parts that are in general quite useful without being too complex to parse. This means we ignore things like generics and such.

The main points of interest of these macros are their `matchers`.
The transcribers are only there for example purposes and are usually not that impressive.

## Function

```rust
macro_rules! function_item_matcher {
    (

        $( #[$meta:meta] )*
    //  ^~~~attributes~~~~^
        $vis:vis fn $name:ident ( $( $arg_name:ident : $arg_ty:ty ),* $(,)? )
    //                          ^~~~~~~~~~~~~~~~argument list~~~~~~~~~~~~~~~^
            $( -> $ret_ty:ty )?
    //      ^~~~return type~~~^
            { $($tt:tt)* }
    //      ^~~~~body~~~~^
    ) => {
        $( #[$meta] )*
        $vis fn $name ( $( $arg_name : $arg_ty ),* ) $( -> $ret_ty )? { $($tt)* }
    }
}

# function_item_matcher!(
#     #[inline]
#     #[cold]
#     pub fn foo(bar: i32, baz: i32, ) -> String {
#         format!("{} {}", bar, baz)
#     }
# );
#
# fn main() {
#     assert_eq!(foo(13, 37), "13 37");
# }
```

A simple function matcher that ignores qualifiers like `unsafe`, `async`, ... as well as generics and where clauses.
If parsing those is required it is likely that you are better off using a proc-macro instead.

This lets you for example, inspect the function signature, generate some extra things from it and then re-emit the entire function again.
Kind of like a `Derive` proc-macro but weaker and for functions.

> Ideally we would like to use a pattern fragment specifier instead of an ident for the arguments but this is currently not allowed.
> Fortunately people don't use non-identifier patterns in function signatures that often so this is okay(a shame, really).

### Method

The macro for parsing basic functions is nice and all, but sometimes we would like to also parse methods, functions that refer to their object via some form of `self` usage. This makes things a bit trickier:

```rust
macro_rules! method_item_matcher {
    // self
    (
        $( #[$meta:meta] )*
    //  ^~~~attributes~~~~^
        $vis:vis fn $name:ident ( $self:ident $(, $arg_name:ident : $arg_ty:ty )* $(,)? )
    //                          ^~~~~~~~~~~~~~~~~~~~~argument list~~~~~~~~~~~~~~~~~~~~~~^
            $( -> $ret_ty:ty )?
    //      ^~~~return type~~~^
            { $($tt:tt)* }
    //      ^~~~~body~~~~^
    ) => {
        $( #[$meta] )*
        $vis fn $name ( $self $(, $arg_name : $arg_ty )* ) $( -> $ret_ty )? { $($tt)* }
    };

    // mut self
    (
        $( #[$meta:meta] )*
        $vis:vis fn $name:ident ( mut $self:ident $(, $arg_name:ident : $arg_ty:ty )* $(,)? )
            $( -> $ret_ty:ty )?
            { $($tt:tt)* }
    ) => {
        $( #[$meta] )*
        $vis fn $name ( mut $self $(, $arg_name : $arg_ty )* ) $( -> $ret_ty )? { $($tt)* }
    };

    // &self
    (
        $( #[$meta:meta] )*
        $vis:vis fn $name:ident ( & $self:ident $(, $arg_name:ident : $arg_ty:ty )* $(,)? )
            $( -> $ret_ty:ty )?
            { $($tt:tt)* }
    ) => {
        $( #[$meta] )*
        $vis fn $name ( & $self $(, $arg_name : $arg_ty )* ) $( -> $ret_ty )? { $($tt)* }
    };

    // &mut self
    (
        $( #[$meta:meta] )*
        $vis:vis fn $name:ident ( &mut $self:ident $(, $arg_name:ident : $arg_ty:ty )* $(,)? )
            $( -> $ret_ty:ty )?
            { $($tt:tt)* }
    ) => {
        $( #[$meta] )*
        $vis fn $name ( &mut $self $(, $arg_name : $arg_ty )* ) $( -> $ret_ty )? { $($tt)* }
    }
}

# struct T(i32);
# impl T {
#     method_item_matcher!(
#         #[inline]
#         pub fn s(self, x: i32) -> String { format!("{}", x) }
#     );
#     method_item_matcher!(
#         pub fn ms(mut self, x: i32,) -> String { format!("{}", x) }
#     );
#     method_item_matcher!(
#         pub fn rs(&self, x: i32, y: i32) -> String { format!("{}", self.0 + x + y) }
#     );
#     method_item_matcher!(
#         pub fn rms(&mut self) -> String { self.0.to_string() }
#     );
# }
#
# fn main() {
#     assert_eq!({ let t = T(11); t.s(11) }, "11");
#     assert_eq!({ let t = T(22); t.ms(22) }, "22");
#     assert_eq!({ let t = T(30); t.rs(1, 2) }, "33");
#     assert_eq!({ let mut t = T(44); t.rms() }, "44");
# }
```
The four rules are identical except for the `self` receiver on both sides of the rule, which is `self`, `mut self`, `&self`, and `&mut self`.
You might not need all four rules.

`$self:ident` must be used in the matcher instead of a bare `self`.
Without that, uses of `self` in the body will cause compile errors, because a macro invocation can only access identifiers it receives from parameters.

## Struct

```rust
macro_rules! struct_item_matcher {
    // Unit-Struct
    (
        $( #[$meta:meta] )*
    //  ^~~~attributes~~~~^
        $vis:vis struct $name:ident;
    ) => {
        $( #[$meta] )*
        $vis struct $name;
    };

    // Tuple-Struct
    (
        $( #[$meta:meta] )*
    //  ^~~~attributes~~~~^
        $vis:vis struct $name:ident (
            $(
                $( #[$field_meta:meta] )*
    //          ^~~~field attributes~~~~^
                $field_vis:vis $field_ty:ty
    //          ^~~~~~a single field~~~~~~^
            ),*
        $(,)? );
    ) => {
        $( #[$meta] )*
        $vis struct $name (
            $(
                $( #[$field_meta] )*
                $field_vis $field_ty
            ),*
        );
    };

    // Named-Struct
    (
        $( #[$meta:meta] )*
    //  ^~~~attributes~~~~^
        $vis:vis struct $name:ident {
            $(
                $( #[$field_meta:meta] )*
    //          ^~~~field attributes~~~!^
                $field_vis:vis $field_name:ident : $field_ty:ty
    //          ^~~~~~~~~~~~~~~~~a single field~~~~~~~~~~~~~~~^
            ),*
        $(,)? }
    ) => {
        $( #[$meta] )*
        $vis struct $name {
            $(
                $( #[$field_meta] )*
                $field_vis $field_name : $field_ty
            ),*
        }
    }
}

# struct_item_matcher!(
#     #[derive(Copy, Clone)]
#     pub(crate) struct Foo {
#        pub bar: i32,
#        baz: &'static str,
#        qux: f32
#     }
# );
# struct_item_matcher!(
#     #[derive(Copy, Clone)]
#     pub(crate) struct Bar;
# );
# struct_item_matcher!(
#     #[derive(Clone)]
#     pub(crate) struct Baz (i32, pub f32, String);
# );
# fn main() {
#     let _: Foo = Foo { bar: 42, baz: "macros can be nice", qux: 3.14, };
#     let _: Bar = Bar;
#     let _: Baz = Baz(2, 0.1234, String::new());
# }
```

# Enum

Parsing enums is a bit more complex than structs so we will finally make use of some of the [patterns] we have discussed, [Incremental TT Muncher] and [Internal Rules].
Instead of just building the parsed enum again we will merely visit all the tokens of the enum, as rebuilding the enum would require us to collect all the parsed tokens temporarily again via a [Push Down Accumulator].

```rust
macro_rules! enum_item_matcher {
    // tuple variant
    (@variant $variant:ident (
        $(
            $( #[$field_meta:meta] )*
    //      ^~~~field attributes~~~~^
            $field_vis:vis $field_ty:ty
    //      ^~~~~~a single field~~~~~~^
        ),* $(,)?
    //∨~~rest of input~~∨
    ) $(, $($tt:tt)* )? ) => {

        // process rest of the enum
        $( enum_item_matcher!(@variant $( $tt )*) )?
    };
    // named variant
    (@variant $variant:ident {
        $(
            $( #[$field_meta:meta] )*
    //      ^~~~field attributes~~~!^
            $field_vis:vis $field_name:ident : $field_ty:ty
    //      ^~~~~~~~~~~~~~~~~a single field~~~~~~~~~~~~~~~^
        ),* $(,)?
    //∨~~rest of input~~∨
    } $(, $($tt:tt)* )? ) => {
        // process rest of the enum
        $( enum_item_matcher!(@variant $( $tt )*) )?
    };
    // unit variant
    (@variant $variant:ident $(, $($tt:tt)* )? ) => {
        // process rest of the enum
        $( enum_item_matcher!(@variant $( $tt )*) )?
    };
    // trailing comma
    (@variant ,) => {};
    // base case
    (@variant) => {};
    // entry point
    (
        $( #[$meta:meta] )*
        $vis:vis enum $name:ident {
            $($tt:tt)*
        }
    ) => {
        enum_item_matcher!(@variant $($tt)*)
    };
}

# enum_item_matcher!(
#     #[derive(Copy, Clone)]
#     pub(crate) enum Foo {
#         Bar,
#         Baz,
#     }
# );
# enum_item_matcher!(
#     #[derive(Copy, Clone)]
#     pub(crate) enum Bar {
#         Foo(i32, f32),
#         Bar,
#         Baz(),
#     }
# );
# enum_item_matcher!(
#     #[derive(Clone)]
#     pub(crate) enum Baz {}
# );
```

[patterns]: ../patterns.md
[Push Down Accumulator]: ../patterns/push-down-acc.md
[Internal Rules]: ../patterns/internal-rules.md
[Incremental TT Muncher]: ../patterns/tt-muncher.md
