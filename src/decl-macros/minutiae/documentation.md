# Documentation

When documenting a declarative macro, be aware that the contents of the macro's matchers are considered public.

For example, a macro like:
```rust
/// Perform an operation.
#[macro_export]
macro_rules! operation {
    (
        // Add two numbers.
        $a:literal + $b:literal
    ) => {
        $a + $b
    };

    (
        // Add a number to itself the given number of times.
        $n:literal * $times:literal
    ) => {
        // For xyz reason, we multiply with an added sum
        // instead of using the built-in operator.
        //
        // Note that this comment is not shown!
        (0..$b).map(|_| $a).sum()
    };
}
```

Shows up in `rustdoc`'s output as something like:

> ### Macro `operation`
> [Source](#documentation)
>
> ```rust
> macro_rules! operation {
>     (
>         // Add two numbers.
>         $a:literal + $b:literal
>     ) => { ... };
>     (
>         // Add a number to itself the given number of times.
>         $n:literal * $times:literal
>     ) => { ... };
> }
> ```
>
> Perform an operation.

For this reason, you might want to avoid [internal rules](../patterns/internal-rules.md) in your top-level macro, and instead prefer to define a private helper macro.
