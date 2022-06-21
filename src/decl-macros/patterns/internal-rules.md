# Internal Rules

```rust
#[macro_export]
macro_rules! foo {
    (@as_expr $e:expr) => {$e};

    ($($tts:tt)*) => {
        foo!(@as_expr $($tts)*)
    };
}
#
# fn main() {
#     assert_eq!(foo!(42), 42);
# }
```

Internal rules can be used to unify multiple `macro_rules!` macros into one, or to make it easier to read and write [TT Munchers] by explicitly naming what rule you wish to call in a macro.

So why is it useful to unify multiple mbe macros into one?
The main reasoning for this is how they are handled in the 2015 Edition of Rust due to `macro_rules!` macros not being namespaced in said edition.
This gives one the troubles of having to re-export all the internal `macro_rules!` macros as well polluting the global macro namespace or even worse, macro name collisions with other crates.
In short, it's quite a hassle.
This fortunately isn't really a problem anymore nowadays with a rustc version >= 1.30, for more information consult the [Import and Export chapter](../minutiae/import-export.html).

Nevertheless, let's talk about how we can unify multiple `macro_rules!` macros into one with this technique and what exactly this technique even is.

We have two `macro_rules!` macros, the common [`as_expr!` macro](../building-blocks/ast-coercion.html) and a `foo` macro that makes use of the first one:

```rust
#[macro_export]
macro_rules! as_expr { ($e:expr) => {$e} }

#[macro_export]
macro_rules! foo {
    ($($tts:tt)*) => {
        as_expr!($($tts)*)
    };
}
#
# fn main() {
#     assert_eq!(foo!(42), 42);
# }
```

This is definitely not the nicest solution we could have for this macro, as it pollutes the global macro namespace as mentioned earlier.
In this specific case `as_expr` is also a very simple macro that we only used once, so let's "embed" this macro in our `foo` macro with internal rules!
To do so, we simply prepend a new matcher for our macro, which consists of the matcher used in the `as_expr` macro, but with a small addition.
We prepend a tokentree that makes it match only when specifically asked to.
In this case we can for example use `@as_expr`, so our matcher becomes `(@as_expr $e:expr) => {$e};`.
With this we get the macro that was defined at the very top of this page:

```rust
#[macro_export]
macro_rules! foo {
    (@as_expr $e:expr) => {$e};

    ($($tts:tt)*) => {
        foo!(@as_expr $($tts)*)
    };
}
#
# fn main() {
#     assert_eq!(foo!(42), 42);
# }
```

You see how we embedded the `as_expr` macro in the `foo` one?
All that changed is that instead of invoking the `as_expr` macro, we now invoke `foo` recursively but with a special token tree prepended to the arguments, `foo!(@as_expr $($tts)*)`.
If you look closely you might even see that this pattern can be combined quite nicely with [TT Munchers]!

The reason for using `@` was that, as of Rust 1.2, the `@` token is *not* used in prefix position; as such, it cannot conflict with anything.
This reasoning became obsolete later on when in Rust 1.7 macro matchers got future proofed by emitting a warning to prevent certain tokens from being allowed to follow certain fragments[^ambiguity-restrictions], which in Rust 1.12 became a hard-error.
There other symbols or unique prefixes may be used as desired, but use of `@` has started to become widespread, so using it may aid readers in understanding your macro.

[^ambiguity-restrictions]:[ambiguity-restrictions](../minutiae/metavar-and-expansion.html)

> **Note**: in the early days of Rust the `@` token was previously used in prefix position to denote a garbage-collected pointer, back when the language used sigils to denote pointer types.
> Its only *current* purpose is for binding names to patterns.
> For this, however, it is used as an *infix* operator, and thus does not conflict with its use here.

Additionally, internal rules will often come *before* any "bare" rules, to avoid issues with `macro_rules!` incorrectly attempting to parse an internal invocation as something it cannot possibly be, such as an expression.

## Performance

One downside of internal rules is that they can hurt compile times.
Only one macro rule can match any (valid) macro invocation, but the compiler must try to match all rules in order.
If a macro has many rules, there can be many such failures, and the use of internal rules will increase the number of such failures.

Also, the `@as_expr`-style identifier makes rules longer, slightly increasing
the amount of work the compiler must do when matching.

Therefore, for best performance, avoiding internal rules is best.
Avoiding them often makes complex macros easier to read, too.

[TT Munchers]:./tt-muncher.html
