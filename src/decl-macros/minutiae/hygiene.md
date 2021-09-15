# Hygiene

`macro_rules!` macros in Rust are *partially* hygienic, also called mixed hygiene.
Specifically, they are hygienic when it comes to *local variables*, *labels* and `$crate`, but nothing else.

Hygiene works by attaching an invisible "syntax context" value to all identifiers.
When two identifiers are compared, *both* the identifiers' textual names *and* syntax contexts must be identical for the two to be considered equal.

To illustrate this, consider the following code:

<pre class="rust rust-example-rendered"><span class="synctx-0"><span class="hljs-built_in">macro_rules</span><span class="hljs-built_in">!</span> using_a {&#xa;    ($e:expr) =&gt; {&#xa;        {&#xa;            <span class="hljs-keyword">let</span> a = <span class="hljs-number">42</span>;&#xa;            $e&#xa;        }&#xa;    }&#xa;}&#xa;&#xa;<span class="hljs-keyword">let</span> four = <span class="hljs-built_in">using_a!</span>(a / <span class="hljs-number">10</span>);</span></pre>

We will use the background colour to denote the syntax context.
Now, let's expand the macro invocation:

<pre class="rust rust-example-rendered"><span class="synctx-0"><span class="hljs-keyword">let</span> four = </span><span class="synctx-1">{&#xa;    <span class="hljs-keyword">let</span> a <span class="op">=</span> <span class="hljs-number">42</span>;&#xa;    </span><span class="synctx-0">a / <span class="hljs-number">10</span></span><span class="synctx-1">&#xa;}</span><span class="synctx-0">;</span></pre>

First, recall that `macro_rules!` invocations effectively *disappear* during expansion.

Second, if you attempt to compile this code, the compiler will respond with something along the following lines:

```text
error[E0425]: cannot find value `a` in this scope
  --> src/main.rs:13:21
   |
13 | let four = using_a!(a / 10);
   |                     ^ not found in this scope
```

Note that the background colour (*i.e.* syntax context) for the expanded macro *changes* as part of expansion.
Each `macro_rules!` macro expansion is given a new, unique syntax context for its contents.
As a result, there are *two different `a`s* in the expanded code: one in the first syntax context, the second in the other.
In other words, <code><span class="synctx-0">a</span></code> is not the same identifier as <code><span class="synctx-1">a</span></code>, however similar they may appear.

That said, tokens that were substituted *into* the expanded output *retain* their original syntax context (by virtue of having been provided to the macro as opposed to being part of the macro itself).
Thus, the solution is to modify the macro as follows:

<pre class="rust rust-example-rendered"><span class="synctx-0"><span class="hljs-built_in">macro_rules!</span> using_a {&#xa;    ($a:ident, $e:expr) =&gt; {&#xa;        {&#xa;            <span class="hljs-keyword">let</span> $a = <span class="hljs-number">42</span>;&#xa;            $e&#xa;        }&#xa;    }&#xa;}&#xa;&#xa;<span class="hljs-keyword">let</span> four = <span class="hljs-built_in">using_a!</span>(a, a / <span class="hljs-number">10</span>);</span></pre>

Which, upon expansion becomes:

<pre class="rust rust-example-rendered"><span class="synctx-0"><span class="hljs-keyword">let</span> four = </span><span class="synctx-1">{&#xa;    <span class="hljs-keyword">let</span> </span><span class="synctx-0">a</span><span class="synctx-1"> = <span class="hljs-number">42</span>;&#xa;    </span><span class="synctx-0">a / <span class="hljs-number">10</span></span><span class="synctx-1">&#xa;}</span><span class="synctx-0">;</span></pre>

The compiler will accept this code because there is only one `a` being used.

### `$crate`

Hygiene is also the reason that we need the `$crate` metavariable when our macro needs access to other items in the defining crate.
What this special metavariable does is that it expands to an absolute path to the defining crate.

```rust,ignore
//// Definitions in the `helper_macro` crate.
#[macro_export]
macro_rules! helped {
    // () => { helper!() } // This might lead to an error due to 'helper' not being in scope.
    () => { $crate::helper!() }
}

#[macro_export]
macro_rules! helper {
    () => { () }
}

//// Usage in another crate.
// Note that `helper_macro::helper` is not imported!
use helper_macro::helped;

fn unit() {
   // but it still works due to `$crate` properly expanding to the crate path `helper_macro`
   helped!();
}
```

Note that, because `$crate` refers to the current crate, it must be used with a fully qualified module path when referring to non-macro items:

```rust
pub mod inner {
    #[macro_export]
    macro_rules! call_foo {
        () => { $crate::inner::foo() };
    }

    pub fn foo() {}
}
```
