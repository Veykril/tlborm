# Push-down Accumulation

```rust
macro_rules! init_array {
    (@accum (0, $_e:expr) -> ($($body:tt)*))
        => {init_array!(@as_expr [$($body)*])};
    (@accum (1, $e:expr) -> ($($body:tt)*))
        => {init_array!(@accum (0, $e) -> ($($body)* $e,))};
    (@accum (2, $e:expr) -> ($($body:tt)*))
        => {init_array!(@accum (1, $e) -> ($($body)* $e,))};
    (@accum (3, $e:expr) -> ($($body:tt)*))
        => {init_array!(@accum (2, $e) -> ($($body)* $e,))};
    (@as_expr $e:expr) => {$e};
    [$e:expr; $n:tt] => {
        {
            let e = $e;
            init_array!(@accum ($n, e.clone()) -> ())
        }
    };
}

let strings: [String; 3] = init_array![String::from("hi!"); 3];
# assert_eq!(format!("{:?}", strings), "[\"hi!\", \"hi!\", \"hi!\"]");
```

All macros in Rust **must** result in a complete, supported syntax element (such as an expression,
item, *etc.*). This means that it is impossible to have a macro expand to a partial construct.

One might hope that the above example could be more directly expressed like so:

```ignore
macro_rules! init_array {
    (@accum 0, $_e:expr) => {/* empty */};
    (@accum 1, $e:expr) => {$e};
    (@accum 2, $e:expr) => {$e, init_array!(@accum 1, $e)};
    (@accum 3, $e:expr) => {$e, init_array!(@accum 2, $e)};
    [$e:expr; $n:tt] => {
        {
            let e = $e;
            [init_array!(@accum $n, e)]
        }
    };
}
```

The expectation is that the expansion of the array literal would proceed as follows:

```rust,ignore
            [init_array!(@accum 3, e)]
            [e, init_array!(@accum 2, e)]
            [e, e, init_array!(@accum 1, e)]
            [e, e, e]
```

However, this would require each intermediate step to expand to an incomplete expression. Even
though the intermediate results will never be used *outside* of a macro context, it is still
forbidden.

Push-down, however, allows us to incrementally build up a sequence of tokens without needing to
actually have a complete construct at any point prior to completion. In the example given at the
top, the sequence of macro invocations proceeds as follows:

```rust,ignore
init_array! { String:: from ( "hi!" ) ; 3 }
init_array! { @ accum ( 3 , e . clone (  ) ) -> (  ) }
init_array! { @ accum ( 2 , e.clone() ) -> ( e.clone() , ) }
init_array! { @ accum ( 1 , e.clone() ) -> ( e.clone() , e.clone() , ) }
init_array! { @ accum ( 0 , e.clone() ) -> ( e.clone() , e.clone() , e.clone() , ) }
init_array! { @ as_expr [ e.clone() , e.clone() , e.clone() , ] }
```

As you can see, each layer adds to the accumulated output until the terminating rule finally emits
it as a complete construct.

The only critical part of the above formulation is the use of `$($body:tt)*` to preserve the output
without triggering parsing. The use of `($input) -> ($output)` is simply a convention adopted to
help clarify the behavior of such macros.

Push-down accumulation is frequently used as part of
[incremental TT munchers](./tt-muncher.md), as it allows arbitrarily complex intermediate
results to be constructed. [Internal Rules](./internal-rules.md) were of use
here as well, as they simplify creating such macros.
