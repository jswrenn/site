+++
title = "A Gentle Introduction to Transmutation Stability"
date = 2020-09-02
tags = ["rust","safer transmutation"]
+++

What's up with transmutation stability?

<!-- more -->

Imagine a trait "`TransmuteFrom`" that is implemented by the compiler for all types `Src` and `Dst` if the bits of `Src` are reinterpretable as `Dst`.

Such a trait would be unusual in two respects:
  - The type system is reasoning about in-memory layouts. Type systems don't *typically* worry about such things.
  - You don't *ask* for this trait to be implemented for your types.

In this post, we'll focus on that second point of weirdness. Most traits are implemented *by request*. If an implementation of a trait exists for a type, it's usually because somebody, *somewhere* has typed the keyword `impl`.

But not in this case: whether our `TransmuteFrom` trait is implemented for a `Src` and `Dst` depends *solely* on the safety of that conversion—not because `impl` was written somewhere.

With traits that are manually implemented, you can trust that an `impl` you are relying on won't simply vanish. In other words, trait implementations are *stable*. Unfortunately, that won't be the case with our transmutation trait, because some of the factors which would affect transmutability are not covered by the existing stability rules.

For instance, let's say a crate defines:
```rust
#[repr(C)]
pub struct Foo {
  pub bar: u8,
  pub baz: u16,
}
```
The current stability rules permit this crate's author to reverse the definition order of the `bar` and `baz` fields in a minor version release, without breaking stability.

Yet doing this *could* affect transmutability. The previous definition of `Foo` is `TransmuteFrom` the type `Fizz`:
```rust
#[repr(C)] pub struct Fizz(i8, i16);
```
...but it won't be if the ordering of `Foo`'s fields is reversed (since that would expose a padding byte as initialized memory).


## Why should the compiler should *automatically* conjure up trait implementations, anyways!?
Alternatives in which the compiler does *not* conjur up trait implementations automatically do not suffer from this weirdness. In the approach taken by the [`FromBits` pre-RFC](https://internals.rust-lang.org/t/pre-rfc-frombits-intobits/7071), `Foo`'s author would simply declare that `Fizz` is transmutable into a type `Foo` by writing a normal `impl`:
```rust
impl FromBits<Fizz> for Foo {};
```
Having provided this `impl`, *removing* it would be considered a breaking change according to the established rules of stability. In other words: no stability weirdness here! Yet, treating transmutability as a normal trait comes at a steep cost: `Foo`'s author must foresee and write an `impl` of `FromBits` for `Foo`for *every* source type from which `Foo` should be reinterpretable.

This degree of manual labor isn't satisfactory. Our RFC and [prior art]((https://github.com/jswrenn/rfcs/blob/safer-transmute/text/0000-safer-transmute.md#automatic) ) address this issue by having the compiler automatically infer implementations of a transmutability trait on-the-fly.

However, whereas [prior art](https://github.com/jswrenn/rfcs/blob/safer-transmute/text/0000-safer-transmute.md#stability-hazards) mostly ignores the hazard this stability weirdness poses, our RFC neutralizes it *almost completely*.

## Mitigating the Stability Weirdness

The Safer Transmute RFC neutralizes this stability weirdness in two ways.

### Make the Weirdness Obvious

First, we make this weirdness **unmissable**. Our transmutation trait *only* behaves unstably if you write the words `NeglectStability`, like this:
```rust
where
    Dst: TransmuteFrom<Src, NeglectStability>
```
(You could also imagine achieving this by naming our transmutation trait something stark like `UnstableTransmuteFrom`. However, our `TransmuteFrom` trait is going to have an extra parameter for neglecting static checks anyways, so we simply use it to indicate that stability is being neglected.)

### Make the Weirdness Opt-In

Second, we make this weirdness **opt-in**. Whereas this:
```rust
where
    Dst: TransmuteFrom<Src, NeglectStability>
```
*doesn't* follow the usual rules of trait implementation of stability, writing *this* **will**:
```rust
where
    Dst: TransmuteFrom<Src>
```
By default, `TransmuteFrom` is *only* implemented for `Src` and `Dst` if it *sound*, *safe* and *stable*. Neglecting any of *any* of these requires passing an additional type parameter to `TransmuteFrom`.

If you don't `NeglectStability`, the transmutability trait follows the all of the usual rules of trait stability. But how? We achieve this *without* any additional specialized compiler machinery, and *without* introducing any new stability rules. Instead, we introduce two normal traits which implemented **manually** by rustaceans to publicly signal that `impl`s of `TransmuteFrom` for their type adhere to the usual trait stability rules.

My next blog post will provide a gentle introduction to exactly how this is accomplished!

