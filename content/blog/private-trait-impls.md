+++
title = "Private Implementations of a Public Trait"
date = 2020-10-18
tags = ["rust"]
+++

In Rust, there is no such thing as a *private* implementation of a public trait. If you implement a public trait for your public type, anyone can take advantage of that implementation! In other words, while types and traits have a first-order notion of visibility, trait *implementations* merely inherit the visibility of the involved trait and types. A third-party cannot use the functionality of a private trait on a public type, nor use the functionality of a public trait on a private type.

So, to make a public trait implementation private, we merely need to introduce a private type into it!

<!-- more -->

To illustrate, consider this public trait, `CTF`:
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

As I hinted, we need to introduce a private type into the implementation for `i8`, but there isn't anywhere for another type to go. So, let's make one:
```rust
pub trait CTF<Scope> {
    fn ctf() {
        println!("You captured the flag!")
    }
}
```
...and modify our implementations to suit. Our implementation of `CTF` for `i8` should be usable *anywhere*:
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

## Ergonomics
Okay, but doesn't this mean we need to *always* supply a scope parameter when invoking `ctf`? Not necessarily. We can simply defined a default value for `Scope`:
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

## Resilience
You *only* need to reason about the visibility of your scope types as defined by their declaration site. Even if you employ the pub-in-priv trick to "smuggle" a scope type out beyond its declared boundary, Rust will forbid you from using it:
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

fn main() {
    use crate_a::{CTF, smuggle::Here};
    
    <u32 as CTF>::ctf(); // Ok!
    
    <u32 as CTF<Here>>::ctf(); // ERROR!
}
```
```
warning: private type `crate_a::Here` in public interface (error E0446)
  --> src/main.rs:22:9
   |
22 |         pub type Here = super::Here;
   |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: `#[warn(private_in_public)]` on by default
   = warning: this was previously accepted by the compiler but is being phased out; it will become a hard error in a future release!
   = note: for more information, see issue #34537 <https://github.com/rust-lang/rust/issues/34537>

error: type `crate_a::Here` is private
  --> src/main.rs:31:5
   |
31 |     <u32 as CTF<Here>>::ctf(); // ERROR!
   |     ^^^^^^^^^^^^^^^^^^^^^^^ private type

error: aborting due to previous error; 1 warning emitted
```

[**Try it for yourself!**](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2018&gist=c711d84abd5187cdb52d90d603601da6)

We owe this resilience thanks to Rust's [*type privacy*](https://github.com/rust-lang/rfcs/blob/master/text/2145-type-privacy.md) guarantees.