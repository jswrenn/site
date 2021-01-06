+++
title = "Rust's SemVer Snares: Alignment"
date = 2021-01-06
tags = ["rust","semver"]
+++
[*(Part of an ongoing series!)*](/blog/semver-snares)

In Rust, changes to a type's alignment are not usually understood to be Breaking Changes™. Of course, that isn't to say you *can't* break downstream code by changing the alignment of a type...

<!-- more -->

## `align_of`
The [`mem::align_of`](https://doc.rust-lang.org/std/mem/fn.align_of.html) intrinsic is a safe function that provides the minimum alignment (in bytes) of any type. As with `size_of`, downstream code should not rely on `mem::align_of` producing a SemVer stable result, but that's only a convention. Consider:
```rust
pub mod upstream {
  #[repr(C)]
  pub struct Foo {
    bar: u8,
    // uncommenting this field is a breaking change for `downstream`:
    /* baz: u16 */
  }
}

pub mod downstream {
  use super::upstream::*;
  
  const _: [(); 1] = [(); std::mem::align_of::<Foo>()];
}
```
<pre class="language-rust_errors"><code class="language-rust_errors"><span class="token error">error<a class="token error-explanation" href="https://doc.rust-lang.org/stable/error-index.html#E0308" target="_blank">[E0308]</a>: mismatched types</span>
  <a class="token error-location" href="#" data-line="12" data-col="22">--&gt; src/lib.rs:12:22
</a>   |
12 |   const _: [(); 1] = [(); std::mem::align_of::&lt;Foo&gt;()];
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected an array with a fixed size of 1 element, found one with 2 elements
</code></pre>

## `repr(transparent)`
The `repr(transparent)` attribute also provides a mechanism for observing the alignment of a type. The `repr(transparent)` attribute can be applied to types where:
- at most one field has size greater-than zero, and
- all *other* fields have minimum alignment equal to 1

...to specify that the annotated type's layout is identical to that of the non-zero-sized field. Applying `repr(transparent)` to a type with more than one field that has alignment >1 is a compile error:
```rust
#[repr(transparent)]
pub struct Foo {
    bar: u8,      // align = 1
    baz: [u16; 0] // align = 2 (⚠)
}
```

<pre class="language-rust_errors"><code class="language-rust_errors"><span class="token error">error<a class="token error-explanation" href="https://doc.rust-lang.org/stable/error-index.html#E0691" target="_blank">[E0691]</a>: zero-sized field in transparent struct has alignment larger than 1</span>
 <a class="token error-location" href="#" data-line="4" data-col="5">--&gt; src/lib.rs:4:5
</a>  |
4 |     baz: [u16; 0]
  |     ^^^^^^^^^^^^^ has alignment larger than 1
</code></pre>

This requirement exists because even zero-sized fields affect the alignment (and thus padding) of the types they appear in.

Consequently, upstream changes that increase the alignment of ZST can break downstream code:

```rust
pub mod upstream {
  #[repr(C)]
  pub struct Foo {
    bar: (),
    // uncommenting this field is a breaking change for `downstream`:
    /* baz: [u16; 0], */
  }
}

pub mod downstream {
  use super::upstream::*;

  #[repr(transparent)]
  struct Bar(u8, Foo);
}
```
<pre class="language-rust_errors"><code class="language-rust_errors"><span class="token error">error<a class="token error-explanation" href="https://doc.rust-lang.org/stable/error-index.html#E0691" target="_blank">[E0691]</a>: zero-sized field in transparent struct has alignment larger than 1</span>
  <a class="token error-location" href="#" data-line="14" data-col="18">--&gt; src/lib.rs:14:18
</a>   |
14 |   struct Bar(u8, Foo);
   |                  ^^^ has alignment larger than 1
</code></pre>

You should therefore avoid `#[repr(transparent)]` unless the ZST field types are *documented* to remain ZSTs.

---

Thanks to [/u/CUViper](https://www.reddit.com/user/CUViper) for [nudging me](https://www.reddit.com/r/rust/comments/kr41sq/semver_snares_sizedness_and_size/gibo0pk/?context=1) towards thinking about alignment-related snares!

