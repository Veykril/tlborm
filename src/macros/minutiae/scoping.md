# Scoping

The way in which function-like macros are scoped can be somewhat unintuitive. They use two forms
of scopes: textual scope, and path-based scope.

When such a macro is invoked by an unqualified identifier(an identifier that isn't part of a
mulit-part  path), it is first looked up in textual scoping and then in path-based scoping should
the first lookup not yield any results. If it is invoked by a qualified identifier it will skip the
textual scoping lookup and instead only do a look up in the path-based scoping.

## Textual Scope

Firstly, unlike everything else in the language, function-like macros will remain visible in
sub-modules.

```rust
macro_rules! X { () => {}; }
mod a {
    X!(); // defined
}
mod b {
    X!(); // defined
}
mod c {
    X!(); // defined
}
# fn main() {}
```

> **Note**: In these examples, remember that all of them have the *same behavior* when the module
> contents are in separate files.

Secondly, *also* unlike everything else in the language, `macro_rules!` macros are only accessible
*after* their definition. Also note that this example demonstrates how `macro_rules!` macros do not
"leak" out of their defining scope:

```rust
mod a {
    // X!(); // undefined
}
mod b {
    // X!(); // undefined
    macro_rules! X { () => {}; }
    X!(); // defined
}
mod c {
    // X!(); // undefined
}
# fn main() {}
```

To be clear, this lexical order dependency applies even if you move the macro to an outer scope:

```rust
mod a {
    // X!(); // undefined
}
macro_rules! X { () => {}; }
mod b {
    X!(); // defined
}
mod c {
    X!(); // defined
}
# fn main() {}
```

However, this dependency *does not* apply to macros themselves:

```rust
mod a {
    // X!(); // undefined
}
macro_rules! X { () => { Y!(); }; }
mod b {
    // X!(); // defined, but Y! is undefined
}
macro_rules! Y { () => {}; }
mod c {
    X!(); // defined, and so is Y!
}
# fn main() {}
```

Defining `macro_rules!` macros multiple times is allowed and the most recent declaration will simply
shadow previous ones unless it has gone out of scope.

```rust
macro_rules! X { (1) => {}; }
X!(1);
macro_rules! X { (2) => {}; }
// X!(1); // Error: no rule matches `1`
X!(2);

mod a {
    macro_rules! X { (3) => {}; }
    // X!(2); // Error: no rule matches `2`
    X!(3);
}
// X!(3); // Error: no rule matches `3`
X!(2);

```

`macro_rules!` macros can be exported from a module using the `#[macro_use]` attribute. Using this
on a module is similar to saying that you do not want to have the module's macro's scope end with
the module.

```rust
mod a {
    // X!(); // undefined
}
#[macro_use]
mod b {
    macro_rules! X { () => {}; }
    X!(); // defined
}
mod c {
    X!(); // defined
}
# fn main() {}
```

Note that this can interact in somewhat bizarre ways due to the fact that identifiers in a
`macro_rules!` macro (including other macros) are only resolved upon expansion:

```rust
mod a {
    // X!(); // undefined
}
#[macro_use]
mod b {
    macro_rules! X { () => { Y!(); }; }
    // X!(); // defined, but Y! is undefined
}
macro_rules! Y { () => {}; }
mod c {
    X!(); // defined, and so is Y!
}
# fn main() {}
```

Another complication is that `#[macro_use]` applied to an `extern crate` *does not* behave this way:
such declarations are effectively *hoisted* to the top of the module. Thus, assuming `X!` is defined
in an external crate called `mac`, the following holds:

```rust,ignore
mod a {
    // X!(); // defined, but Y! is undefined
}
macro_rules! Y { () => {}; }
mod b {
    X!(); // defined, and so is Y!
}
#[macro_use] extern crate macs;
mod c {
    X!(); // defined, and so is Y!
}
# fn main() {}
```

Finally, note that these scoping behaviors apply to *functions* as well, with the exception of
`#[macro_use]` (which isn't applicable):

```rust
macro_rules! X {
    () => { Y!() };
}

fn a() {
    macro_rules! Y { () => {"Hi!"} }
    assert_eq!(X!(), "Hi!");
    {
        assert_eq!(X!(), "Hi!");
        macro_rules! Y { () => {"Bye!"} }
        assert_eq!(X!(), "Bye!");
    }
    assert_eq!(X!(), "Hi!");
}

fn b() {
    macro_rules! Y { () => {"One more"} }
    assert_eq!(X!(), "One more");
}
# 
# fn main() {
#     a();
#     b();
# }
```

These scoping rules are why a common piece of advice is to place all `macro_rules!` macros which
should be accessible "crate wide" at the very top of your root module, before any other modules.
This ensures they are available *consistently*. This also applies to `mod` definitions for files, as
in:

```rs
#[macro_use]
mod some_mod_that_defines_macros;
mod some_mod_that_uses_those_macros;
```

The order here is important, swap the declaration order and it won't compile.

## Path-Based Scope

By default, a `macro_rules!` macro has no path-based scope. However, if it has the `#[macro_export]`
attribute, then it is declared in the crate root scope and can be referred to similar to how you
refer to any other item. The [Import and Export] chapter goes more in-depth into said attribute.

[Import and Export]: ./import-export.html