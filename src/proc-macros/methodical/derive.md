# Derive

Derive procedural macros define new inputs for the [`derive`](https://doc.rust-lang.org/reference/attributes/derive.html) attribute.
This type can be invoked by feeding it to a derive attribute's input, e.g. `#[derive(TlbormDerive)]`.

A simple skeleton of a derive procedural macro looks like the following:
```rs
use proc_macro::TokenStream;

#[proc_macro_derive(TlbormDerive)]
pub fn tlborm_derive(item: TokenStream) -> TokenStream {
    TokenStream::neW()
}
```

The `proc_macro_derive` is a bit more special in that it requires an extra identifier, this identifier will become the actual name of the derive proc macro.
The input token stream is the item the derive attribute is attached to, that is, it will always be an `enum`, `struct` or `union` as these are the only items a derive attribute can annotated.
The returned token stream will be **appended** to the containing block or module of the annotated item with the requirement that the token stream consists of a set of valid items.

Usage example:
```rs
use tlborm_proc::TlbormDerive;

#[derive(TlbormDerive)]
struct Foo;
```

### Helper Attributes

Derive proc macros are a bit more special in that they can add additional attributes visible only in the scope of the item definition.
These attributes are called *derive macro helper attributes* and are [inert](https://doc.rust-lang.org/reference/attributes.html#active-and-inert-attributes).
Their purpose is to give derive proc macros additional customizability on a per field or variant basis, that is these attributes can be used to annotate fields or enum variants while having no effect on their own.
As they are `inert` they will not be stripped and are visible to all macros.

They can be defined by adding an `attributes(helper0, helper1, ..)` argument to the `proc_macro_derive` attribute containing a comma separated list of identifiers which are the names of the helper attributes.

Thus a simple skeleton of a derive procedural macro with helper attributes looks like the following:
```rs
use proc_macro::TokenStream;

#[proc_macro_derive(TlbormDerive, attributes(tlborm_helper))]
pub fn tlborm_derive(item: TokenStream) -> TokenStream {
    TokenStream::neW()
}
```

That is all there is to helper attributes, to consume them in the proc macro the implementation will then have to check the attributes of fields and variants to see whether they are attributed with the corresponding helper.
It is an error to use a helper attribute if none of the used derive macros of the given item declare it as such, as the compiler will then instead try to resolve it as a normal attribute.

Usage example:
```rs
use tlborm_proc::TlbormDerive;

#[derive(TlbormDerive)]
struct Foo {
    #[tlborm_helper]
    field: u32
}

#[derive(TlbormDerive)]
enum Bar {
    #[tlborm_helper]
    Variant { #[tlborm_helper] field: u32 }
}
```
