# Attribute

Attribute procedural macros define new *outer* attributes which can be attached to items.
This type can be invoked with the `#[attr]` or `#[attr(…)]` syntax where `…` is an arbitrary token tree.

A simple skeleton of an attribute procedural macro looks like the following:
```rs
use proc_macro::TokenStream;

#[proc_macro_attribute]
pub fn tlborm_attribute(input: TokenStream, annotated_item: TokenStream) -> TokenStream {
    annotated_item
}
```

Of note here is that unlike the other two procedural macro kinds, this one has two input parameters instead of one.
- The first parameter is the delimited token tree following the attribute's name, excluding the delimiters around it.
It is empty if the attribute is written bare, that is just a name without a `(TokenTree)` following it, e.g. `#[attr]`.
- The second token stream is the item the attribute is attached to *without* the attribute this proc macro defines.
As this is an [`active`](https://doc.rust-lang.org/reference/attributes.html#active-and-inert-attributes) attribute, the attribute will be stripped from the item before it is being passed to the proc macro.

The returned token stream will **replace** the annotated item fully.
Note that the replacement does not have to be a single item, it can be 0 or more.
<!-- CONFIRM: Is this true? Can it emit an empty token stream? -->

Usage example:
```rs
use tlborm_proc::tlborm_attribute;

#[tlborm_attribute]
fn foo() {}

#[tlborm_attribute(attributes are pretty handsome)]
fn bar() {}
```
