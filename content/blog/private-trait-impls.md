+++
title = "Scoped Trait Implementations"
date = 2020-10-18
tags = ["rust"]
+++

In Rust, the `impl` keyword doesn't have an associated visibility; there's no such thing as `pub impl`. However, that isn't to say that implementations don't have visibility—they do! Enter: [**RFC2145 - Type Privacy**](https://github.com/rust-lang/rfcs/pull/2145).

<!-- more -->

RFC2145 formalizes the notion of "private implementations" in terms of Rust's existing visibility mechanisms. In short, you don't *need* `pub impl` or `pub(self) impl` to figure out who can see a trait implementation—you can do so by inspecting the visibility of its components. Specifically: a trait implementation is as visible as the *least* visible of its trait, target type, or type parameters.

## Implementation Privacy By Example
Given this:
```rust
/* This type is private to this module. */
enum Private {}

/* This type is public outside of this module. */
pub enum Public {}

/* This type is public outside of this module */
pub trait Trait<Parameter> {
    fn function();
}
```

...this is a **public** impl:
```rust
impl<Anything> Trait<Anything> for Public {
    fn function() {
        println!("Callable anywhere.");
    }
}
```

...this is a **private** impl:
```rust
impl<Anything> Trait<Anything> for Private {
    fn function() {
        println!("Callable only where `Private` is in scope.");
    }
}
```

...and this is a **private** impl:
```rust
impl<Anything> Trait<Private> for Anything {
    fn function() {
        println!("Callable only where `Private` is in scope.");
    }
}
```

Type privacy [prohibits the use of associated items from private impls](https://github.com/petrochenkov/rfcs/blob/9901089b2467b0813456479ba03642e9f0ac0912/text/0000-type-privacy.md#user-content-additional-restrictions-for-associated-items:~:text=type%20privacy%20additionally%20prohibits%20use%20of%20any%20associated%20items%20from%20private%20impls), so whether `function` is publicly callable depends on the visibility of the implementation.

## Putting Type Privacy to Work
We can leverage these rules to create "hidden implementations" of public traits. To illustrate, consider this public trait, `CTF`:
```rust
pub trait CTF {
    fn ctf() {
        println!("You captured the flag!")
    }
}

impl CTF for u32 {}

impl CTF for i8 {}
```
How can we make our implementation of `CTF` for `u32` available to everyone, but our implementation for `i8` only available to us?

Per RFC2145, these two trait implementations are public: `CTF` is a public trait and `i8` is a public type. To make these trait implementations private, we need only add a type parameter to `CTF`:
```rust
pub trait CTF<Scope> {
    fn ctf() {
        println!("You captured the flag!")
    }
}
```
...and modify our implementations to suit.

Our implementation of `CTF` for `i8` should be usable *anywhere*:
```rust
impl<Anywhere> CTF<Anywhere> for u32 {}
```

And our implementation of `CTF` for `i8` should only be usable *here*:
```rust
/* a private type! */
enum Here {}

impl CTF<Here> for u32 {}
```
**That's it!**

### Ergonomics
Okay, but doesn't this mean we need to *always* supply a scope parameter when invoking `ctf` and modify *all* of our existing public implementations of the trait? No! We can simply define a *public* default value for `Scope`:
```rust
pub trait CTF<Scope = ()> {
    fn ctf() {
        println!("You captured the flag!")
    }
}
```
...so third-parties can still just write:
```rust
<u32 as CTF>::ctf();
```

...and existing implementations can remain unchanged; e.g.:
```rust
impl CTF for Foo {}
//      ^ don't need to provide `Scope` parameter
```

### Resilience

If you are relying on type privacy for safety, it's crucial that your private types (like `Here`) remain private. Fortunately, you can do so *solely* by manually examining their definition site. While it is possible to publicly re-export private types via public type aliases:
```rust
pub mod crate_a {

    enum Here {}

    pub trait CTF<Scope=()> {
        fn ctf();
    }

    impl<Anywhere> CTF<Anywhere> for u32 {
        fn ctf() {
            println!("Anyone can capture the flag for u32!")
        }
    }

    impl CTF<Here> for i8 {
        fn ctf() {
            println!("Only things that can see `Here` can capture the flag for i8!");
        }
    }

    pub mod smuggle {
        pub type Here = super::Here;
    }
}
```
...these cases are *automatically* linted:
<pre class="  language-rust_errors"><code class="language-rust_errors"><span class="token warning">warning: private type `crate_a::Here` in public interface (error E0446)</span>
  <a class="token error-location" data-line="21" data-col="5">--&gt; src/lib.rs:21:5
</a>   |
21 |     pub type Here = Here;
   |     ^^^^^^^^^^^^^^^^^^^^^
   |
<span class="token note">   = note: `#[warn(private_in_public)]` on by default</span>
   = warning: this was previously accepted by the compiler but is being phased out; it will become a hard error in a future release!
<span class="token note">   = note: for more information, <a class="token see-issue" href="https://github.com/rust-lang/rust/issues/34537" target="_blank">see issue #34537 &lt;https://github.com/rust-lang/rust/issues/34537&gt;</a></span>
</code></pre>

And **using** such smuggled types:
```rust
fn main() {
    use crate_a::{CTF, smuggle::Here};
    
    <u32 as CTF>::ctf(); // Ok!
    
    <u32 as CTF<Here>>::ctf(); // ERROR!
}
```
...produces hard errors:
<pre class="  language-rust_errors"><code class="  language-rust_errors"><span class="token error">error: type `crate_a::Here` is private</span>
  <a class="token error-location" href="#" data-line="31" data-col="5">--&gt; src/main.rs:31:5
</a>   |
31 |     &lt;u32 as CTF&lt;Here&gt;&gt;::ctf(); // ERROR!
   |     ^^^^^^^^^^^^^^^^^^^^^^^ private type
</code></pre>

[**Try it for yourself!**](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2018&gist=c711d84abd5187cdb52d90d603601da6)

## A future for `pub impl`?

There *has* been some prior work to add a `pub(...) impl` syntax to Rust (including [this yet-to-be-merged RFC](https://github.com/rust-lang/rfcs/pull/2529)), but defining extensions to the trait system is a thorny endeavor. However, these proposals have not (to my knowledge) taken advantage of Rust's existing type privacy work. If they did, perhaps we could reason about and experiment with the implications of such proposals *within* Rust's *current* trait system.

Concretely: Any public trait can be made "scope-aware" by adding a `Scope` parameter to it; e.g.:
```rust
pub trait Default<Scope=()> {
    fn default() -> Self;
}
```
Then, we might think of `pub impl Default for Foo` as desugaring to:
```rust
impl<Anywhere> Default<Anywhere> for Foo {
    fn default() -> Self { ... }
}
```
...and think of `pub(self) impl Default for Foo` as desugaring to:
```rust
pub(self) enum Here {}

impl Default<Here> for Foo {
    fn default() -> Self { ... }
}
```

This is only a sketch, but it might be a promising starting point for a native `pub(...) impl` syntax!
