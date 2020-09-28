# Counting

## Repetition with replacement

Counting things in a macro is a surprisingly tricky task. The simplest way is to use replacement
with a repetition match.

```rust
macro_rules! replace_expr {
    ($_t:tt $sub:expr) => {$sub};
}

macro_rules! count_tts {
    ($($tts:tt)*) => {0usize $(+ replace_expr!($tts 1usize))*};
}
# 
# fn main() {
#     assert_eq!(count_tts!(0 1 2), 3);
# }
```

This is a fine approach for smallish numbers, but will likely *crash the compiler* with inputs of
around 500 or so tokens. Consider that the output will look something like this:

```rust,ignore
0usize + 1usize + /* ~500 `+ 1usize`s */ + 1usize
```

The compiler must parse this into an AST, which will produce what is effectively a perfectly
unbalanced binary tree 500+ levels deep.

## Recursion

An older approach is to use recursion.

```rust
macro_rules! count_tts {
    () => {0usize};
    ($_head:tt $($tail:tt)*) => {1usize + count_tts!($($tail)*)};
}
# 
# fn main() {
#     assert_eq!(count_tts!(0 1 2), 3);
# }
```

> **Note**: As of `rustc` 1.2, the compiler has *grievous* performance problems when large numbers
> of integer literals of unknown type must undergo inference. We are using explicitly
> `usize`-typed literals here to avoid that.
>
> If this is not suitable (such as when the type must be substitutable), you can help matters by
> using `as` (*e.g.* `0 as $ty`, `1 as $ty`, *etc.*).

This *works*, but will trivially exceed the recursion limit. Unlike the repetition approach, you can
extend the input size by matching multiple tokens at once.

```rust
macro_rules! count_tts {
    ($_a:tt $_b:tt $_c:tt $_d:tt $_e:tt
     $_f:tt $_g:tt $_h:tt $_i:tt $_j:tt
     $_k:tt $_l:tt $_m:tt $_n:tt $_o:tt
     $_p:tt $_q:tt $_r:tt $_s:tt $_t:tt
     $($tail:tt)*)
        => {20usize + count_tts!($($tail)*)};
    ($_a:tt $_b:tt $_c:tt $_d:tt $_e:tt
     $_f:tt $_g:tt $_h:tt $_i:tt $_j:tt
     $($tail:tt)*)
        => {10usize + count_tts!($($tail)*)};
    ($_a:tt $_b:tt $_c:tt $_d:tt $_e:tt
     $($tail:tt)*)
        => {5usize + count_tts!($($tail)*)};
    ($_a:tt
     $($tail:tt)*)
        => {1usize + count_tts!($($tail)*)};
    () => {0usize};
}

fn main() {
    assert_eq!(700, count_tts!(
        ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
        ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
        
        ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
        ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
        
        ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
        ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
        
        ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
        ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
        
        ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
        ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
        
        // Repetition breaks somewhere after this
        ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
        ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,

        ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
        ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,, ,,,,,,,,,,
    ));
}
```

This particular formulation will work up to ~1,200 tokens.

## Slice length

A third approach is to help the compiler construct a shallow AST that won't lead to a stack overflow.
This can be done by constructing an array literal and calling the `len` method.

```rust
macro_rules! replace_expr {
    ($_t:tt $sub:expr) => {$sub};
}

macro_rules! count_tts {
    ($($tts:tt)*) => {<[()]>::len(&[$(replace_expr!($tts ())),*])};
}
# 
# fn main() {
#     assert_eq!(count_tts!(0 1 2), 3);
# }
```

This has been tested to work up to 10,000 tokens, and can probably go much higher.

## Enum counting

This approach can be used where you need to count a set of mutually distinct identifiers.

```rust
macro_rules! count_idents {
    ($($idents:ident),* $(,)*) => {
        {
            #[allow(dead_code, non_camel_case_types)]
            enum Idents { $($idents,)* __CountIdentsLast }
            const COUNT: u32 = Idents::__CountIdentsLast as u32;
            COUNT
        }
    };
}
# 
# fn main() {
#     const COUNT: u32 = count_idents!(A, B, C);
#     assert_eq!(COUNT, 3);
# }
```

This method does have two drawbacks. First, as implied above, it can *only* count valid identifiers
(which are also not keywords), and it does not allow those identifiers to repeat.

Secondly, this approach is *not* hygienic, meaning that if whatever identifier you use in place of
`__CountIdentsLast` is provided as input, the macro will fail due to the duplicate variants in the
`enum`.

## Bit twiddling

Another recursive approach using bit operations: 

```rust
macro_rules! count_tts {
    () => { 0 };
    ($odd:tt $($a:tt $b:tt)*) => { (count_tts!($($a)*) << 1) | 1 };
    ($($a:tt $even:tt)*) => { count_tts!($($a)*) << 1 };
}
# 
# fn main() {
#     assert_eq!(count_tts!(0 1 2), 3);
# }
```

This approach is pretty smart as it effectively halves its input whenever its even and then
multiplying the counter by 2 (or in this case shifting 1 bit to the left which is equivalent). If
the input is uneven it simply takes one token tree from the input `or`s the token tree to the
previous counter which is equivalent to adding 1 as the lowest bit has to be a 0 at this point due
to the previous shifting. Rinse and repeat until we hit the base rule `() => 0`.

The benefit of this is that the constructed AST expression that makes up the counter value will grow
with a complexity of `O(log(n))` instead of `O(n)` like the other approaches. Be aware that you can
still hit the recursion limit with this if you try hard enough. Credits for this method go to Reddit
user [`YatoRust`](https://www.reddit.com/r/rust/comments/d3yag8/the_little_book_of_rust_macros/).


Let's go through the procedure by hand once:

```rust,ignore
count_tts!(0 0 0 0 0 0 0 0 0 0);
```
This invocation will match the third rule due to the fact that we have an even number of token trees
(10). The matcher names the odd token trees in the sequence `$a` and the even ones `$even` but the
expansion only makes use of `$a`, which means it effectively discards all the even elements cutting
the input in half. So the invocation now becomes:
```rust,ignore
count_tts!(0 0 0 0 0) << 1;
```
This invocation will now match the second rule as its input is an uneven amount of token trees. In
this case the first token tree is discarded to make the input even again, then we also do the
halving step in this invocation again since we know the input would be even now anyways. Therefor we
can count 1 for the uneven discard and multiply by 2 again since we also halved.
```rust,ignore
((count_tts!(0 0) << 1) | 1) << 1;
```
```rust,ignore
((count_tts!(0) << 1 << 1) | 1) << 1;
```
```rust,ignore
(((count_tts!() | 1) << 1 << 1) | 1) << 1;
```
```rust,ignore
((((0 << 1) | 1) << 1 << 1) | 1) << 1;
```

Now to check if we expanded correctly manually we can use a one of the tools we introduced for
[`debugging`](/macros/minutiae/debugging.html). When expanding the macro there we should get:
```rust,ignore
((((0 << 1) | 1) << 1 << 1) | 1) << 1;
```

That's the same so we didn't make any mistakes, great!