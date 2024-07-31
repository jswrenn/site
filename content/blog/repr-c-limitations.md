+++
title = "`repr(C)`: Clear, Simple and (Sometimes) Wrong"
date = 2024-07-31
tags = ["Rust"]
+++

Too often, `repr(C)` is thought of as a panacea to layout stability and portability problems; that values of a type marked `repr(C)` can be consistently reflected upon, across different platforms, different compiler versions, and even perhaps different minor library versions. That is not always the case!

<!-- more -->

The promise of `repr(C)` is quite limited: applied to a struct, it guarantees that a particular layout algorithm will be used for that struct. That algorithm uses the definition order, sizes, and alignments of its fields as inputs. When those inputs change, the output — the struct's layout — may change, too.

This post highlights two ways in which `repr(C)` may fall short of expectations.

## `repr(C)` layouts are not always platform portable

As their names suggest, the sizes of Rust's primitive integer types are well-specified. A `u8` is always 8 bits; an `i128` is always 128 bits. By contrast, their alignments are unspecified. A `u128` can have an alignment anywhere between 1 and 16!

Consequently, `repr(C)` alone does not guarantee that a struct has a consistent alignment. For example:
```rust
/// The size of this struct is 16 bytes across all toolchains and targets, but
/// its alignment is unspecified. On some existing targets the alignment is 4, on others the alignment is 8.
#[repr(C)]
struct AlignUnportable(u128);
```
...nor does it guarantee that a struct will have a consistent size:
```rust
/// Both the size and alignment of this struct are unspecified. Although the
/// `u8` field will always appear at byte offset 0, a variable amount of of trailing
/// padding will be added depending on the alignment of `AlignUnportable`.
#[repr(C)]
struct SizeUnportable(
    [AlignUnportable; 0],
    u8,
);
```
...nor does it guarantee that a struct's fields will have consistent offsets:
```rust
/// Not only are both the size and alignment of this struct unspecified, the
/// byte offset of the `u8` field is also unspecified.
#[repr(C)]
struct OffsetUnportable(
    AlignUnportable,
    u8,
);
```

These inconsistencies are relevant whenever you transmuting a Rust struct into bytes on one computer and transmuting those bytes back into a Rust struct on a different computer.

Safe transmutation crates like [zerocopy](https://crates.io/crates/zerocopy) and [bytemuck](https://crates.io/crates/bytemuck) do not yet ensure portability. Although these crates will prevent unportable transmutations from inducing undefined behavior, compilation errors and unexpected runtime behavior might still arise.

Stay tuned for developments in this space. In zerocopy, we're planning to add both [inline layout assertions](https://github.com/google/zerocopy/issues/1329) and a [marker trait for portability](https://github.com/google/zerocopy/issues/1262). In the mean time, use [`offset_of!`](https://doc.rust-lang.org/nightly/core/mem/macro.offset_of.html) and the [static_assertions](https://crates.io/crates/static_assertions) crate to test that your layouts match your expectations.

## `repr(C)` layouts are not always SemVer stable

The limitations of `repr(C)` are not only technical, but also social. The central social contract of Rust's crate authors is SemVer, a convention to consider some changes to be "major" (like removing an API) and others as "minor" (like adding an API), and to adjust version numbers accordingly. The line between major and minor changes is defined by [*RFC 1105: API Evolution*](https://rust-lang.github.io/rfcs/1105-api-evolution.html).

This document says *nothing* about `repr(C)`. No official documentation says anything about the SemVer contract of `repr(C)`. Is it acceptable to freely add and remove `repr(C)` between minor versions? Is it acceptable to reorder public fields of `repr(C)` structs? Is it acceptable to shift the offsets of public fields of `repr(C)` structs?

These questions are wholly unanswered. If you require these guarantees from a library, you should request that they are explicitly documented by the library.

## Safety Goggles for Alchemists

If this post intrigued you, consider attending my upcomming RustConf talk, [*Safety Goggles for Alchemists*](https://rustconf.com/programs/#676). In this talk, you’ll learn how Rust is poised to become the first systems programming language with transmutation safety, and how safe transmute is already being put to use to build next-gen systems.
