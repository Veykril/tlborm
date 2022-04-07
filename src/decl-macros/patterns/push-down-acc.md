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

All syntax extensions in Rust **must** result in a complete, supported syntax element (such as an expression, item, *etc.*).
This means that it is impossible to have a syntax extension expand to a partial construct.

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

However, this would require each intermediate step to expand to an incomplete expression.
Even though the intermediate results will never be used *outside* of a macro context, it is still forbidden.

Push-down, however, allows us to incrementally build up a sequence of tokens without needing to actually have a complete construct at any point prior to completion.
In the example given at the top, the sequence of invocations proceeds as follows:

```rust,ignore
init_array! { String:: from ( "hi!" ) ; 3 }
init_array! { @ accum ( 3 , e . clone (  ) ) -> (  ) }
init_array! { @ accum ( 2 , e.clone() ) -> ( e.clone() , ) }
init_array! { @ accum ( 1 , e.clone() ) -> ( e.clone() , e.clone() , ) }
init_array! { @ accum ( 0 , e.clone() ) -> ( e.clone() , e.clone() , e.clone() , ) }
init_array! { @ as_expr [ e.clone() , e.clone() , e.clone() , ] }
```

As you can see, each layer adds to the accumulated output until the terminating rule finally emits it as a complete construct.

The only critical part of the above formulation is the use of `$($body:tt)*` to preserve the output without triggering parsing.
The use of `($input) -> ($output)` is simply a convention adopted to help clarify the behavior of such macros.

Push-down accumulation is frequently used as part of [incremental TT munchers](./tt-muncher.md), as it allows arbitrarily complex intermediate results to be constructed.
[Internal Rules](./internal-rules.md) were of use here as well, as they simplify creating such macros.

## Performance

Push-down accumulation is inherently quadratic.
Consider a push-down accumulation rule that builds up an accumulator of 100 token trees, one token tree per invocation.
- The initial invocation will match against the empty accumulator.
- The first recursive invocation will match against the accumulator of 1 token tree.
- The next recursive invocation will match against the accumulator of 2 token trees.

And so on, up to 100.
This is a classic quadratic pattern, and long inputs can cause macro expansion to blow out compile times.
Furthermore, TT munchers are also inherently quadratic over their input, so a macro that uses both TT munching *and* push-down accumulation will be doubly quadratic!

All the [performance advice](./tt-muncher.md#performance) about TT munchers holds for push-down accumulation. 
In general, avoid using them too much, and keep them as simple as possible.

Finally, make sure you put the accumulator at the *end* of rules, rather than the beginning.
That way, if a rule fails, the compiler won't have had to match the (potentially long) accumulator before hitting the part of the rule that fails to match.
This can make a large difference to compile times.

