+++
title = "Rust's SemVer Snares: `repr(transparent)` Super-Cut"
date = 2021-01-09
tags = ["rust","semver"]
+++
[*(Part of an ongoing series!)*](/blog/semver-snares)

In the last two posts, `repr(transparent)` provided an unusual mechanism by which safe, downstream code could inadvertently rely on the [size](/blog/semver-snares-size/) and [alignment](/blog/semver-snares-alignment/) of an upstream type. In this post, I'll recap the issue and discuss why it is tricky to fix.

<!-- more -->

To recap, `repr(transparent)` attribute provides a mechanism for observing the alignment of a type. The `repr(transparent)` attribute can be applied to types where:
- at most one field has size greater-than zero, and
- all *other* fields have minimum alignment equal to 1

...to specify that the annotated type's layout is identical to that of the non-zero-sized field.

Applying `repr(transparent)` to a type with more than one field of size ≥1 is a compile error:
```rust
#[repr(transparent)]
pub struct Foo {
    bar: u8, // size = 1
    baz: u8  // size = 1 (⚠)
}
```
...as is applying `repr(transparent)` to a type with more than one field having alignment >1:
```rust
#[repr(transparent)]
pub struct Foo {
    bar: u8,      // align = 1
    baz: [u16; 0] // align = 2 (⚠)
}
```

**At present, you should not use `#[repr(transparent)]` unless you are sure your ZST fields are *guaranteed* to have size equal to zero, and alignment equal to one.**

What would it take to fully shift this diligence from the programmer to the compiler?

## Potentially-Breaking Changes
To answer this, let's consider the sorts of (otherwise) non-breaking changes that can increase the alignment and size of ZSTs.

### `repr` Annotations
Applying a `repr` annotation to a type can alter its size and alignment. The `align(N)` modifier specifies that the annotated type will have a minimum alignment of at least `N`:
```rust
// uncommenting this line breaks `Bar`:
/* #[repr(align(2))] */
struct Foo;

#[repr(transparent)]
struct Bar(u8, Foo);
```

Adding `repr(C)` or `repr(<primitive>)` to an ZST `enum` can increase its size:
```rust
// uncommenting this line breaks `Bar`:
/* #[repr(isize)] */
enum Foo {
  Variant
}

#[repr(transparent)]
struct Bar(u8, Foo);
```

### Fields
Generally speaking, it is not a breaking change to:
- modify or remove private fields
- add private fields to structs marked with `#[non_exhaustive]`
- add private fields to structs that already have private fields

...but, in the presence of `repr(transparent)`, all of the above changes can potentially break downstream code.

The minimum alignment of `repr(C)` and `repr(transparent)` types is equal to the greatest minimum alignment of its fields. Adding a >1-aligned field to a 1-aligned ZST prohibits that ZST from use as a field in a `repr(transparent)` type:
```rust
#[repr(C)]
pub struct Foo {
    bar: [u8; 0], // align == 1
    // uncommenting this field breaks `Bar`:
    /* baz: [u16; 0], */ // align == 2
}

#[repr(transparent)]
struct Bar(u8, Foo);
```
Likewise, adding or modifying a field of a ZST such that the size increases in a breaking change:
```rust
#[repr(C)]
pub struct Foo {
    bar: (),
    // uncommenting this field breaks `Bar`:
    /* baz: u8 */
}

#[repr(transparent)]
struct Bar(u8, Foo);
```

### Type Parameters
As type parameters provide a mechanism for consumers to alter the private, internal details of a type, *changes* to how type parameters are instantiated directly effect the alignment of a type:
```rust
/// `Foo` is *always* a ZST, but its alignment is equal to that of `T` 
#[repr(C)] struct Foo<T>([T; 0]);

assert_eq!(0, size_of::<Foo<u8>>());
assert_eq!(0, size_of::<Foo<u16>>());

assert_eq!(1, align_of::<Foo<u8>>());
assert_eq!(2, align_of::<Foo<u16>>());
```
...and the size of a type:
```rust
/// `Foo` is 1-aligned, but has the size of `T`
#[repr(C, packed)] struct Foo<T>(MaybeUninit<T>);

assert_eq!(1, size_of::<Foo<u8>>());
assert_eq!(2, size_of::<Foo<u16>>());

assert_eq!(1, align_of::<Foo<u8>>());
assert_eq!(1, align_of::<Foo<u16>>());
```

Consequently, Rust must usually assume that generically-instantiated fields are *not* one-aligned ZSTs:
```rust
#[repr(transparent)]
struct Foo<T, U>(T, [U; 0]);
```
<pre class="language-rust_errors"><code class="language-rust_errors"><span class="token error">error<a class="token error-explanation" href="https://doc.rust-lang.org/nightly/error-index.html#E0690" target="_blank">[E0690]</a>: transparent struct needs exactly one non-zero-sized field, but has 2</span>
 <a class="token error-location" href="#" data-line="2" data-col="1">--&gt; src/lib.rs:2:1
</a>  |
2 | struct Foo&lt;T, U&gt;(T, [U; 0]);
  | ^^^^^^^^^^^^^^^^^-^^------^^
  | |                |  |
  | |                |  this field is non-zero-sized
  | |                this field is non-zero-sized
  | needs exactly one non-zero-sized field, but has 2
</code></pre>

This error is necessary if one wants to provide *definition-site* errors for `repr(transparent)` violations.

### Const Parameters
Unsurprisingly, the instantiation of a const-generic parameter can affect the size of a type:
```rust
/// the alignment of `Foo<N>` is 1
/// the size of `Foo<N>` is `N` bytes
#[repr(C)]
struct Foo<const N: usize>([u8; N]);
```
The instantiation of const-generic parameters can also affect the alignment of a type:
```rust
use std::mem::align_of;

/// alignment of `ZST<{N}>` is equal to `N`
/// the size of `ZST<{N}>` is equal to 0.
#[repr(C)]
pub struct ZST<const N: usize>
where
    (): Align<{N}>,
{
    align: [<() as Align<{N}>>::Type; 0],
}

assert_eq!(1, align_of::<ZST<1>>());
assert_eq!(2, align_of::<ZST<2>>());

pub trait Align<const N: usize> { type Type; }
#[repr(align(1))] pub struct Align1;
#[repr(align(2))] pub struct Align2;
/* and so on */
impl Align<{1}> for () { type Type = Align1; }
impl Align<{2}> for () { type Type = Align2; }
/* and so on */
```

### Default `repr` and `rustc` Version
Generally speaking, the layout properties of "default repr" (i.e., a type without a `repr` attribute) are *unspecified*. To my knowledge, it is not currently specified that:
```rust
enum Foo {
  Bar
}
```
is guaranteed to be a one-aligned and zero-sized. Although `Foo` may be laid out as such by *particular* versions of Rust (such as the version available at the time of writing), that may not be true for *future* versions of Rust. This is a deeper issue than just SemVer stability.

## Enforcing SemVer Stability
At the time of writing, there is [some effort to eliminate](https://github.com/rust-lang/rust/issues/78586) the stability hazards of `repr(transparent)`. However, to *comprehensively* enforce that uses of `#[repr(transparent)]` are SemVer-respecting at type definition sites in this manner, `rustc` would need implement all of the following restrictions atop the basic well-formedness check:
1. prohibit, on all but one field, most occurences of type parameters
2. prohibit, on all but one field, most occurences of const parameters
3. require, on all but one field, that field types are fully-implicitly constructible
4. require, on all but one field, that field types are fully-implicitly constructible
5. require, on all but one field, that field types have well-specified sizes and alignments
6. document that changing the `repr` of *any* one-aligned ZST is a SemVer Breaking Change™

These requirements have far-reaching implications for the role of layout and `repr` in SemVer stability. Since these adjustments would likely need to be timed with an edition change anyways, it's worth considering if a simpler formulation of `repr(transparent)` exists. I think there is: limit `repr(transparent)` to structs on which at most one field is *not* `PhantomData`.

## Beyond `repr(transparent)`
SemVer aside, `repr(transparent)`'s restrictions on generic parameters are complex and unwieldy. These restrictions are necessary to ensure, **at definition site**, that *any* instantiation of the annotated type will be transparent with respect to its non-one-aligned-ZST field. But, in the future, `repr(transparent)` may not be necessary at all as a layout modifier (merely as a definition-site check).

The Unsafe Code Working Group [proposes](https://github.com/rust-lang/unsafe-code-guidelines/pull/164) that one-aligned-ZST fields shalt not influence the layout of default-`repr` structs, and that default-`repr` structs with exactly one *non*-one-aligned-ZST field shall have layout identical to that of the field. If accepted, these rules would mean that one could determine whether or not a particular type was *effectively* transparent, even in the absence of `repr(transparent)`. What would be missing is an in-language mechanism to double-check. To this end, I'd suggest the introduction of the compiler-intrinsic trait `mem::AbiEq<Other>`, which is implemented for all types whose ABI is identical to `Other`.
