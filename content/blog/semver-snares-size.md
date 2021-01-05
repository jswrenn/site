+++
title = "Rust's SemVer Snares: Sizedness and Size"
date = 2021-01-05
tags = ["rust","semver"]
+++
[*(Part of an ongoing series!)*](/blog/semver-snares)

In Rust, changes to a type's size are not usually understood to be Breaking Changes™. Of course, that isn't to say you *can't* break downstream code by changing the size of a type...

<!-- more -->

## Sizedness
For one, you can change the *sizedness* of a type, by adding an unsized field:
```rust
pub mod upstream {
  pub struct Foo {
    bar: u8,
    // uncommenting this field is a breaking change:
    /* baz: [u8] */
  }
}

pub mod downstream {
  use super::upstream::*;

  fn example(foo: Foo) {
    todo!()
  }
}
```

<pre class="language-rust_errors"><code class="language-rust_errors"><span class="token error">error<a class="token error-explanation" href="https://doc.rust-lang.org/stable/error-index.html#E0277" target="_blank">[E0277]</a>: the size for values of type `[u8]` cannot be known at compilation time</span>
  <a class="token error-location" href="#" data-line="11" data-col="14">--&gt; src/lib.rs:11:14
</a>   |
11 |   fn example(foo: Foo) {
   |              ^^^ doesn't have a size known at compile-time
   |
   = <span class="token rust-errors-help">help: within `upstream::Foo`, the trait `Sized` is not implemented for `[u8]`
</span><span class="token note">   = note: required because it appears within the type `upstream::Foo`</span>
<span class="token rust-errors-help">help: function arguments must have a statically known size, borrowed types always have a known size
</span>   |
11 |   fn example(&amp;foo: Foo) {
   |              ^
</code></pre>

## Size
Changing the size of a `Sized` type can also break (poorly-behaving) downstream code. The [`mem::size_of`](https://doc.rust-lang.org/core/mem/fn.size_of.html) intrinsic is a safe function that provides the size (in bytes) of any [`Sized`](https://doc.rust-lang.org/core/marker/trait.Sized.html) type. By convention, downstream code should not rely on `mem::size_of` producing a SemVer stable result, but that's only a convention. Consider:
```rust
pub mod upstream {
  pub struct Foo {
    bar: u8,
    // uncommenting this field is a breaking change for `downstream`:
    /* baz: u8 */
  }
}

pub mod downstream {
  use super::upstream::*;
  
  const _: [(); 1] = [(); std::mem::size_of::<Foo>()];
}
```
<pre class="language-rust_errors"><code class="language-rust_errors"><span class="token error">error<a class="token error-explanation" href="https://doc.rust-lang.org/stable/error-index.html#E0308" target="_blank">[E0308]</a>: mismatched types</span>
  <a class="token error-location" href="#" data-line="12" data-col="22">--&gt; src/lib.rs:12:22
</a>   |
12 |   const _: [(); 1] = [(); std::mem::size_of::&lt;Foo&gt;()];
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected an array with a fixed size of 1 element, found one with 2 elements
</code></pre>

## *Zero* Sizedness
A downstream crate author doesn't *only* need to worry that they aren't using `mem::size_of` in a manner that breaks the stability contract of upstream code. As of 2018, there's another mechanism that observes the size of a type: [`#[repr(transparent)]`](https://doc.rust-lang.org/1.26.2/unstable-book/language-features/repr-transparent.html).

The `repr(transparent)` attribute can be applied to types with at most one non-zero-sized field to specify that the annotated type's layout is identical to that of the field. Applying `repr(transparent)` to a type with more than one non-zero-sized field is a compiler error:
```rust
#[repr(transparent)]
pub struct Foo {
    bar: u8,
    baz: u8
}
```

<pre class="  language-rust_errors"><code class="  language-rust_errors"><span class="token error">error<a class="token error-explanation" href="https://doc.rust-lang.org/stable/error-index.html#E0690" target="_blank">[E0690]</a>: transparent struct needs exactly one non-zero-sized field, but has 2</span>
 <a class="token error-location" href="#" data-line="2" data-col="1">--&gt; src/lib.rs:2:1
</a>  |
2 | pub struct Foo {
  | ^^^^^^^^^^^^^^ needs exactly one non-zero-sized field, but has 2
3 |     bar: u8,
  |     ------- this field is non-zero-sized
4 |     baz: u8
  |     ------- this field is non-zero-sized
</code></pre>

Consequently, upstream changes that turn ZSTs into non-ZSTs can break downstream code.

```rust
pub mod upstream {
  pub struct Foo {
    // uncommenting this field is a breaking change for `downstream`:
    /* bar: u8, */
  }
}

pub mod downstream {
  use super::upstream::*;

  #[repr(transparent)]
  struct Bar(u8, Foo);
}
```
<pre class="language-rust_errors"><code class="language-rust_errors"><span class="token error">error<a class="token error-explanation" href="https://doc.rust-lang.org/stable/error-index.html#E0690" target="_blank">[E0690]</a>: transparent struct needs exactly one non-zero-sized field, but has 2</span>
  <a class="token error-location" href="#" data-line="12" data-col="3">--&gt; src/lib.rs:12:3
</a>   |
12 |   struct Bar(u8, Foo);
   |   ^^^^^^^^^^^--^^---^^
   |   |          |   |
   |   |          |   this field is non-zero-sized
   |   |          this field is non-zero-sized
   |   needs exactly one non-zero-sized field, but has 2
</code></pre>

You should therefore avoid `#[repr(transparent)]` unless the ZST field types are *documented* to remain ZSTs.
