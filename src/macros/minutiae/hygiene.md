# Hygiene

Macros in Rust are *partially* hygienic. Specifically, they are hygienic when it comes to most
identifiers, but *not* when it comes to generic type parameters or lifetimes.

Hygiene works by attaching an invisible "syntax context" value to all identifiers. When two
identifiers are compared, *both* the identifiers' textural names *and* syntax contexts must be
identical for the two to be considered equal.

To illustrate this, consider the following code:

```rust
macro_rules! using_a {
    ($e:expr) => {
        {
            let a = 42;
            $e
        }
    }
}

let four = using_a!(a / 10);
```

Now, let's expand the macro invocation:

```rust,ignore
let four = {
    let a = 42;
    a / 10
};
```

First, recall that `macro_rules!` invocations effectively *disappear* during expansion.

Second, if you attempt to compile this code, the compiler will respond with something along the
following lines:

```text
error[E0425]: cannot find value `a` in this scope
  --> src/main.rs:13:21
   |
13 | let four = using_a!(a / 10);
   |                     ^ not found in this scope
```

Each macro expansion is given a new, unique syntax context for its contents. As a result, there are
*two different `a`s* in the expanded code: one in the first syntax context, the second in the other.

That said, tokens that were substituted *into* the expanded output *retain* their original syntax
context (by virtue of having been provided to the macro as opposed to being part of the macro itself).
Thus, the solution is to modify the macro as follows:

```rust
macro_rules! using_a {
    ($a:ident, $e:expr) => {
        {
            let $a = 42;
            $e
        }
    }
}

let four = using_a!(a, a / 10);
```

Which, upon expansion becomes:

```rust,ignore
let four = {
    let a = 42;
    a / 10
};
```

The compiler will accept this code because there is only one `a` being used.
