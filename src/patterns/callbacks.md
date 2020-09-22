# Callbacks

```rust
macro_rules! call_with_larch {
    ($callback:ident) => { $callback!(larch) };
}

macro_rules! expand_to_larch {
    () => { larch };
}

macro_rules! recognise_tree {
    (larch) => { println!("#1, the Larch.") };
    (redwood) => { println!("#2, the Mighty Redwood.") };
    (fir) => { println!("#3, the Fir.") };
    (chestnut) => { println!("#4, the Horse Chestnut.") };
    (pine) => { println!("#5, the Scots Pine.") };
    ($($other:tt)*) => { println!("I don't know; some kind of birch maybe?") };
}

fn main() {
    recognise_tree!(expand_to_larch!());
    call_with_larch!(recognise_tree);
}
```

Due to the order that macros are expanded in, it is (as of Rust 1.2) impossible to pass information
to a macro from the expansion of *another* macro. This can make modularising macros very difficult.

An alternative is to use recursion and pass a callback. Here is a trace of the above example to
demonstrate how this takes place:

```rust,ignore
recognise_tree! { expand_to_larch ! (  ) }
println! { "I don't know; some kind of birch maybe?" }
// ...

call_with_larch! { recognise_tree }
recognise_tree! { larch }
println! { "#1, the Larch." }
// ...
```

Using a `tt` repetition, one can also forward arbitrary arguments to a callback.

```rust
macro_rules! callback {
    ($callback:ident( $($args:tt)* )) => {
        $callback!( $($args)* )
    };
}

fn main() {
    callback!(callback(println("Yes, this *was* unnecessary.")));
}
```

You can, of course, insert additional tokens in the arguments as needed.
