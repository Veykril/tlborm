# Import and Export

There are two ways to expose a macro to a wider scope. The first is the `#[macro_use]` attribute.
This can be applied to *either* modules or external crates.  For example:

```rust
#[macro_use]
mod macros {
    macro_rules! X { () => { Y!(); } }
    macro_rules! Y { () => {} }
}

X!();
#
# fn main() {}
```

Macros can be exported from the current crate using `#[macro_export]`.  Note that this *ignores* all
visibility.

Given the following definition for a library package `macs`:

```rust,ignore
mod macros {
    #[macro_export] macro_rules! X { () => { Y!(); } }
    #[macro_export] macro_rules! Y { () => {} }
}

// X! and Y! are *not* defined here, but *are* exported,
// despite `macros` being private.
```

The following code will work as expected:

```rust,ignore
X!(); // X is defined
#[macro_use] extern crate macs;
X!();
# 
# fn main() {}
```

Note that you can *only* `#[macro_use]` an external crate from the root module.

Finally, when importing macros from an external crate, you can control *which* macros you import.
You can use this to limit namespace pollution, or to override specific macros, like so:

```rust,ignore
// Import *only* the `X!` macro.
#[macro_use(X)] extern crate macs;

// X!(); // X is defined, but Y! is undefined

macro_rules! Y { () => {} }

X!(); // X is defined, and so is Y!

fn main() {}
```

When exporting macros, it is often useful to refer to non-macro symbols in the defining crate.
Because crates can be renamed, there is a special substitution variable available: `$crate`. This
will *always* expand to an absolute path prefix to the containing crate (*e.g.* `:: macs`).

Note that this does *not* work for macros, since macros do not interact with regular name resolution
in any way. That is, you cannot use something like `$crate::Y!` to refer to a particular macro
within your crate. The implication, combined with selective imports via `#[macro_use]` is that there
is currently *no way* to guarantee any given macro will be available when imported by another crate.

It is recommended that you *always* use absolute paths to non-macro names, to avoid conflicts,
*including* names in the standard library.
