# Scoping

The way in which macros are scoped can be somewhat unintuitive. Firstly, unlike everything else in
the language, macros will remain visible in sub-modules.

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

> **Note**: In these examples, remember that all of them have the *same behaviour* when the module
    contents are in separate files.

Secondly, *also* unlike everything else in the language, macros are only accessible *after* their
definition. Also note that this example demonstrates how macros do not "leak" out of their defining
scope:

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

Macros can be exported from a module using the `#[macro_use]` attribute.

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

Note that this can interact in somewhat bizarre ways due to the fact that identifiers in a macro
(including other macros) are only resolved upon expansion:

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

Finally, note that these scoping behaviours apply to *functions* as well, with the exception of
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

These scoping rules are why a common piece of advice is to place all macros which should be
accessible "crate wide" at the very top of your root module, before any other modules. This ensures
they are available *consistently*.
