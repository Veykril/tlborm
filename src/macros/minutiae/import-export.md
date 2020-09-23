# Import and Export

Importing macros differs between the two Rust Editions, 2015 and 2018. It is recommended to read
both parts nevertheless, as the 2018 Edition can still use the constructs that are explained in the
2015 Edition.

## Edition 2015

In Edition 2015 you have to use the `#[macro_use]` attribute that has already been introduced in the
[scoping chapter]. This can be applied to *either* modules or external crates. For
example:

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

Macros can be exported from the current crate using `#[macro_export]`. Note that this *ignores* all
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

This works, as said in the [scoping chapter], because `#[macro_use]` works slightly different on
extern crates, as it basically *hoists* the exported macros out of the crate to the top of the
module.

> Note: you can *only* `#[macro_use]` an external crate from the root module.

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
Because crates can be renamed, there is a special substitution variable available: [`$crate`]. This
will *always* expand to an absolute path prefix to the containing crate (*e.g.* `:: macs`).

Note that this does *not* work for macros, since macros do not interact with regular name resolution
in any way, unless your compiler version is >= 1.30. In that case it works for macros all editions.
Otherwise, you cannot use something like `$crate::Y!` to refer to a particular macro within your
crate. The implication, combined with selective imports via `#[macro_use]` is that there is
currently *no way* to guarantee any given macro will be available when imported by another crate.

It is recommended that you *always* use absolute paths to non-macro names, to avoid conflicts,
*including* names in the standard library.

[`$crate`]:./hygiene.html#crate

## Edition 2018

The 2018 Edition made our lifes a lot easier when it comes to macros. Why you ask? Quite simply
because it managed to make macros feel more like proper items than some special thing in the
language. What this means is that we can properly import and use them in a namespaced fashion!

So instead of using `#[macro_use]` to import every exported macro from a crate into the global
namespace we can now do the following:

```rs
use some_crate::some_macro;

fn main() {
    some_macro!("hello");
    // as well as
    some_crate::some_other_macro!("macro world");
}
```

Unfortunately, this only applies for external crates, if you use macros that you have defined in
your own crate you are still required to go with `#[macro_use]` on the defining modules. So scoping
applies there the same way as before as well.

> The `$crate` prefix works in this version for everything, macros and items alike since this Edition
> came out with Rust 1.31.

[scoping chapter]:./scoping.html