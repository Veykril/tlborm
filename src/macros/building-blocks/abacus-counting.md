# Abacus Counters

> **Provisional**: needs a more compelling example. Matching nested groups that are *not* denoted by
> Rust groups is sufficiently unusual that it may not merit inclusion.

> **Note**: this section assumes understanding of [push-down accumulation](#push-down-accumulation)
> and [incremental TT munchers](#incremental-tt-munchers).

```rust
macro_rules! abacus {
    ((- $($moves:tt)*) -> (+ $($count:tt)*)) => {
        abacus!(($($moves)*) -> ($($count)*))
    };
    ((- $($moves:tt)*) -> ($($count:tt)*)) => {
        abacus!(($($moves)*) -> (- $($count)*))
    };
    ((+ $($moves:tt)*) -> (- $($count:tt)*)) => {
        abacus!(($($moves)*) -> ($($count)*))
    };
    ((+ $($moves:tt)*) -> ($($count:tt)*)) => {
        abacus!(($($moves)*) -> (+ $($count)*))
    };

    // Check if the final result is zero.
    (() -> ()) => { true };
    (() -> ($($count:tt)+)) => { false };
}

fn main() {
    let equals_zero = abacus!((++-+-+++--++---++----+) -> ());
    assert_eq!(equals_zero, true);
}
```

This technique can be used in cases where you need to keep track of a varying counter that starts at
 or near zero, and must support the following operations:

* Increment by one.
* Decrement by one.
* Compare to zero (or any other fixed, finite value).

A value of *n* is represented by *n* instances of a specific token stored in a group. Modifications
are done using recursion and [push-down accumulation](#push-down-accumulation). Assuming the token
used is `x`, the operations above are implemented as follows:

* Increment by one: match `($($count:tt)*)`, substitute `(x $($count)*)`.
* Decrement by one: match `(x $($count:tt)*)`, substitute `($($count)*)`.
* Compare to zero: match `()`.
* Compare to one: match `(x)`.
* Compare to two: match `(x x)`.
* *(and so on...)*

In this way, operations on the counter are like flicking tokens back and forth like an abacus.[^abacus]

[^abacus]: This desperately thin reasoning conceals the *real* reason for this name: to avoid having
    *yet another* thing with "token" in the name. Talk to your writer about avoiding
    [semantic satiation](https://en.wikipedia.org/wiki/Semantic_satiation) today!\
    In fairness, it could *also* have been called ["unary counting"](https://en.wikipedia.org/wiki/Unary_numeral_system).

In cases where you want to represent negative values, *-n* can be represented as *n* instances of a
*different* token. In the example given above, *+n* is stored as *n* `+` tokens, and *-m* is stored
as *m* `-` tokens.

In this case, the operations become slightly more complicated; increment and decrement effectively
reverse their usual meanings when the counter is negative. To which given `+` and `-` for the
positive and negative tokens respectively, the operations change to:

* Increment by one:
  * match `()`, substitute `(+)`.
  * match `(- $($count:tt)*)`, substitute `($($count)*)`.
  * match `($($count:tt)+)`, substitute `(+ $($count)+)`.
* Decrement by one:
  * match `()`, substitute `(-)`.
  * match `(+ $($count:tt)*)`, substitute `($($count)*)`.
  * match `($($count:tt)+)`, substitute `(- $($count)+)`.
* Compare to 0: match `()`.
* Compare to +1: match `(+)`.
* Compare to -1: match `(-)`.
* Compare to +2: match `(++)`.
* Compare to -2: match `(--)`.
* *(and so on...)*

Note that the example at the top combines some of the rules together (for example, it combines
increment on `()` and `($($count:tt)+)` into an increment on `($($count:tt)*)`).

If you want to extract the actual *value* of the counter, this can be done using a regular
[counter macro](./counting.md). For the example above, the terminal rules can be replaced with the following:

```rust,ignore
macro_rules! abacus {
    // ...

    // This extracts the counter as an integer expression.
    (() -> ()) => {0};
    (() -> (- $($count:tt)*)) => {
        - ( count_tts!($( $count_tts:tt )*) )
    };
    (() -> (+ $($count:tt)*)) => {
        count_tts!($( $count_tts:tt )*)
    };
}

// One of the many token tree counting macros in the counting chapter
macro_rules! count_tts {
    // ...
}
```

> **<abbr title="Just for this example">JFTE</abbr>**: strictly speaking, the above formulation of
> `abacus!` is needlessly complex.  It can be implemented much more efficiently using repetition,
> provided you *do not* need to match against the counter's value in a macro:
>
> ```ignore
> macro_rules! abacus {
>     (-) => {-1};
>     (+) => {1};
>     ($( $moves:tt )*) => {
>         0 $(+ abacus!($moves))*
>     }
> }
> ```
