+++
title = "Private Methods on a Public Trait"
date = 2020-10-18
tags = ["rust"]
+++

In Rust, the methods of a trait inherit the visibility of the trait itself:
```rust
pub trait Foo<Arg> {

    pub(self) fn foo(&self, arg: Arg);

    /* other public methods */

}
```
```
error[E0449]: unnecessary visibility qualifier
 --> src/lib.rs:7:3
  |
7 |   pub(self) fn foo(&self, arg: Arg);
  |   ^^^^^^^^^
```
At minimum, we should hide this method from our documentation:
```rust
pub trait Foo<Arg> {

    #[doc(hidden)]
    fn foo(&self, arg: Arg);

    /* other public methods */

}
```
...but this doesn't *actually* prevent outsiders from calling our method! **Fortunately, there are at least three other patterns `#[doc(hidden)]` can be combined with to make `foo` externally unusable.**

<!-- more -->

The first two approaches are well-established in the Rust community, but the third might be relatively novel.

## Private Super-Trait
Since the methods of a trait inherit the visibility of the trait itself, we can simply place our private methods in a private trait! We then make this private trait a super-trait of our public trait:
```rust
pub(crate) mod private {

    #[doc(hidden)]
    pub trait FooPrivate<Arg> {
        fn foo(&self, arg: Arg);
    }

}

pub trait Foo<Arg>: private::FooPrivate<Arg> {

    /* other public methods */

}
```

But wait: `FooPrivate` is marked `pub`! And what's with this new `private` module? Well, Rust usually forbids embedding a private type in a public type signature:
```rust
pub(crate) trait FooPrivate<Arg> {
    fn foo(&self, arg: Arg);
}

pub trait Foo<Arg>: FooPrivate<Arg> {

}
```
```
error[E0445]: crate-visible trait `FooPrivate<Arg>` in public interface
 --> src/lib.rs:5:1
  |
1 |   pub(crate) trait FooPrivate<Arg> {
  |   ---------- `FooPrivate<Arg>` declared as crate-visible
...
5 | / pub trait Foo<Arg>: FooPrivate<Arg> {
6 | |
7 | | }
  | |_^ can't leak crate-visible trait
```

This pattern of making a public type *effectively* private by embedding it in a private module is called the "pub-in-priv trick". This trick is also the foundation of the [sealed traits](https://rust-lang.github.io/api-guidelines/future-proofing.html#sealed-traits-protect-against-downstream-implementations-c-sealed) pattern for exposing public traits that are only privately implementable.

This method is effective, albeit verbose. Of course, you must be careful to not publicly re-export `FooPrivate` from your crate. If `Foo` is a well-established trait and you intend for the addition of `foo` to be non-breaking, you may have a hard time convincing yourself that you can add a super-trait without breaking any downstream code.

## Private Value Argument
Alternatively, we can add an effectively-private type as an argument to `foo`:
```
pub(crate) mod private {
    pub struct Local;
}

pub trait Foo<Arg> {

    #[doc(hidden)]
    fn foo(&self, arg: Arg, _: private::Local);

    /* other public methods */

}
```
Access to `Local` provides the [*capability*](https://en.wikipedia.org/wiki/Capability-based_security) to call `foo`. Consumers of our crate cannot call `foo`, because they cannot construct an instance of `Local`. *Or can they?*

Well, *yes, they can*. Again, if, you publicly leak a value to `private::Local`, e.g.:
```rust
pub const LOCAL: private::Local = private::Local;
```
...then downstream users can simply re-use that value to call `foo`!

And, regardless of how careful *you* are, a dash of `unsafe` renders `foo` callable:
```rust
Foo::foo(some_arg, unsafe { mem::zeroed!() })
```

## Private Type Argument
Fortunately, we can do better. Rather than exploiting the uninstantiability of `Local`, we exploit the *unnamability* of `Local` and make it a *type* argument of `foo`:
```rust
pub(crate) mod private {
    // Once again, we introduce an effectively
    // private type to encode locality.
    // This time, we make it uninhabited so we
    // *cannot* accidentally leak it.
    pub enum Local {}

    // However, we pair it with a 'sealed' trait
    // that is *only* implemented for `Local`.
    pub trait IsLocal {}

    impl IsLocal for Local {}
}

pub trait Foo<Arg> {

    // The only `L` that renders `foo` callable is `Local`
    fn foo<L: private::IsLocal>(&self, arg: Arg);

    /* other public methods */
}
```

Now *we* can call `foo` from within our crate:
```rust
Foo::foo::<private::Local>(some_arg)
```
...but *consumers* of our crate cannot! Not even unsafely — there is no type-level equivalent to `mem::zeroed()` by which consumers can summon the name of `Local`. *Or is there?*

### Resilience
This time, there isn't! Type *inference*, in principal, provides such a mechanism, but Rust does not perform *full-program* type inference — inference is limited to the bodies of functions. With this in mind, let's try to exploit inference within function bodies to call `CTF::ctf()` in `main`:
```rust
pub mod crate_a {

    pub(self) mod private {
        pub enum Local {}

        pub trait IsLocal {}

        impl IsLocal for Local {}
    }

    pub trait CTF {
        fn ctf<L: private::IsLocal>() {
            println!("You captured the flag!")
        }
    }

    impl CTF for usize {}

    // for this to work, we'll need to leak `Local`
    // by putting it into a public type signature.
    pub fn leak_local() -> private::Local {
        loop {}
    }
}

fn main() {
    use crate_a::{CTF, leak_local};

    fn call_ctf<L, F: Fn() -> L>(f: F) {
        usize::ctf::<L>()
    }

    call_ctf(leak_local)
}
```
This produces an error:
```
error[E0277]: the trait bound `L: crate_a::private::IsLocal` is not satisfied
  --> src/main.rs:29:9
   |
12 |         fn ctf<L: private::IsLocal>() {
   |         ----------------------------- required by `crate_a::CTF::ctf`
...
29 |         usize::ctf::<L>()
   |         ^^^^^^^^^^^^^^^ the trait `crate_a::private::IsLocal` is not implemented for `L`
   |
help: consider restricting type parameter `L`
   |
28 |     fn call_ctf<L: crate_a::private::IsLocal, F: Fn() -> L>(f: F) {
   |                  ^^^^^^^^^^^^^^^^^^^^^^^^^^^
```
We *cannot* apply this suggestion: `IsLocal` is effectively private! Of course, our use of the pub-in-priv trick means that its usual caveats apply: if we *directly* publicly re-export `Local` or `IsLocal` from `crate_a` (e.g., via `pub use`), `ctf` loses the guarantees it gained from relying on their privacy.

### Use in the Wild
I recently used this technique in a [PR removing `unsafe` from the typenum crate](https://github.com/paholg/typenum/pull/142) to [add a private method to a long-public trait](https://github.com/paholg/typenum/pull/142#discussion_r396152986). I wanted my PR to preserve the public API surface of typenum (so new public traits or methods!) and to be non-breaking. My caution about accidentally introducing any breaking changes dissuaded me from the first approach (i.e., a new private super-trait). The drawbacks of the second approach (cluttering of call-sites and that its protections can be bypassed with `unsafe` code) lead me to apply the third approach.
