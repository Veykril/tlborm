# Callbacks

Due to the order that macros are expanded in, it is (as of Rust 1.2) impossible to pass information to a macro from the expansion of *another* macro:

```rust
macro_rules! recognize_tree {
    (larch) => { println!("#1, the Larch.") };
    (redwood) => { println!("#2, the Mighty Redwood.") };
    (fir) => { println!("#3, the Fir.") };
    (chestnut) => { println!("#4, the Horse Chestnut.") };
    (pine) => { println!("#5, the Scots Pine.") };
    ($($other:tt)*) => { println!("I don't know; some kind of birch maybe?") };
}

macro_rules! expand_to_larch {
    () => { larch };
}

fn main() {
    recognize_tree!(expand_to_larch!());
    // first expands to:  recognize_tree! { expand_to_larch ! (  ) }
    // and then:          println! { "I don't know; some kind of birch maybe?" }
}
```

This can make modularizing macros very difficult.

An alternative is to use recursion and pass a callback:

```rust
// ...
# macro_rules! recognize_tree {
#     (larch) => { println!("#1, the Larch.") };
#     (redwood) => { println!("#2, the Mighty Redwood.") };
#     (fir) => { println!("#3, the Fir.") };
#     (chestnut) => { println!("#4, the Horse Chestnut.") };
#     (pine) => { println!("#5, the Scots Pine.") };
#     ($($other:tt)*) => { println!("I don't know; some kind of birch maybe?") };
# }

macro_rules! call_with_larch {
    ($callback:ident) => { $callback!(larch) };
}

fn main() {
    call_with_larch!(recognize_tree);
    // first expands to:  call_with_larch! { recognize_tree }
    // then:              recognize_tree! { larch }
    // and finally:       println! { "#1, the Larch." }
}
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
