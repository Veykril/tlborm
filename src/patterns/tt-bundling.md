# TT Bundling

```rust
macro_rules! call_a_or_b_on_tail {
    ((a: $a:ident, b: $b:ident), call a: $($tail:tt)*) => {
        $a(stringify!($($tail)*))
    };

    ((a: $a:ident, b: $b:ident), call b: $($tail:tt)*) => {
        $b(stringify!($($tail)*))
    };

    ($ab:tt, $_skip:tt $($tail:tt)*) => {
        call_a_or_b_on_tail!($ab, $($tail)*)
    };
}

fn compute_len(s: &str) -> Option<usize> {
    Some(s.len())
}

fn show_tail(s: &str) -> Option<usize> {
    println!("tail: {:?}", s);
    None
}

fn main() {
    assert_eq!(
        call_a_or_b_on_tail!(
            (a: compute_len, b: show_tail),
            the recursive part that skips over all these
            tokens doesn't much care whether we will call a
            or call b: only the terminal rules care.
        ),
        None
    );
    assert_eq!(
        call_a_or_b_on_tail!(
            (a: compute_len, b: show_tail),
            and now, to justify the existence of two paths
            we will also call a: its input should somehow
            be self-referential, so let's make it return
            some eighty-six!
        ),
        Some(92)
    );
}
```

In particularly complex recursive macros, a large number of arguments may be needed in order to
carry identifiers and expressions to successive layers. However, depending on the implementation
there may be many intermediate layers which need to forward these arguments, but do not need to
*use* them.

As such, it can be very useful to bundle all such arguments together into a single TT by placing
them in a group. This allows layers which do not need to use the arguments to simply capture and
substitute a single [`tt`], rather than having to exactly capture and substitute the entire argument
group.

The example above bundles the `$a` and `$b` expressions into a group which can then be forwarded as
a single [`tt`] by the recursive rule. This group is then destructured by the terminal rules to access
the expressions.

[`tt`]:./fragment-specifiers.html#tt
