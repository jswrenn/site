+++
title = "Taming nalgebra's Rustdoc"
date = 2020-09-10
tags = ["rust"]
+++

[nalgebra]: https://nalgebra.org/
[`Matrix`]: https://nalgebra.org/rustdoc/nalgebra/base/struct.Matrix.html
[`Vector`]: https://docs.rs/nalgebra/0.22.0/nalgebra/base/type.Vector.html


[Nalgebra][nalgebra] is a powerhouse of functionality, but its documentation can be overwhelming—the [documentation for `Matrix`][`Matrix`] lists over *600* methods. Your documentation endeavors might not be *quite* so overwhelming, but you still could benefit from these **three tricks** nalgebra uses to improve its docs.

<!-- more -->

## Documenting Type Aliases
**TIP: Write `impl`s and documentation on type aliases.**


The [`Matrix`] `struct` type is at the heart of nalgebra's functionality. It is generically parametrized by dimension, so the same outer type is used to encode matrices of all sizes. A vector of dimension `R`, for instance, is merely a `Matrix` of `R` rows and `U1` columns.

From a programming ergonomics perspective, you might think it'd be convenient to codify this with a type alias; e.g.:
```rust
type Vector<N, D, S> = Matrix<N, D, U1, S>;
```
...and you'd be right; nalgebra defines just such a type alias!

But type aliases aren't *just* programming shorthand; they can be used to improve documentation, too: **When an inherent `impl` is written in terms of a type alias, the documentation of that `impl` *also* appears of the documentation page of that type alias.**

Sure enough, if you visit nalgebra's [`Vector` documentation page][`Vector`] type alias, you'll see *only* the methods specific to vectors:

<a style="display:block" href="https://docs.rs/nalgebra/0.22.0/nalgebra/base/type.Vector.html#impl-1"><iframe scrolling="no" style="pointer-events:none; height:90vh; width:100%" src="https://docs.rs/nalgebra/0.22.0/nalgebra/base/type.Vector.html#impl-1"></iframe></a>


Unfortunately, this same documentation is [*also* rendered on the page for `Matrix`](https://docs.rs/nalgebra/0.21.1/nalgebra/base/struct.Matrix.html#impl-1) and *without* the type aliases. This is why the documentation for the base `Matrix` type is so long. :-(

## Coalescing `impl`s
**TIP: Reduce repetition by grouping methods with the same bounds into the a single bounded `impl`.**

One of nalgebra's cooler ergonomic shortcuts is [*vector swizzling*](https://en.wikipedia.org/wiki/Swizzling_(computer_graphics)). A swizzle lets you build a new vector from some ordering and subset of the components of another vector. For instance, `vec.xyx()` constructs a new, three-dimensional vector comprised of the `x`, `y` and `x` components of `vec`.

Supporting this shortcut requires generating a *lot* of methods. A couple years ago, the documentation of these methods looked like this:

<a style="display:block" href="https://docs.rs/nalgebra/0.16.11/nalgebra/base/type.Vector.html#impl-15"><iframe scrolling="no" style="pointer-events:none; height:90vh; width:100%" src="https://docs.rs/nalgebra/0.16.11/nalgebra/base/type.Vector.html#impl-15"></iframe></a>

**This documentation has a very poor signal-to-noise ratio.** The preamble
```rust
impl<N: Scalar, D: DimName, S: Storage<N, D>> Vector<N, D, S>
```
is repeated for *every* swizzling method, and each individual swizzling method has its *own* `where` bound documenting its dimensionality requirements.

Nearly all of this repetition was eliminated with a [minor change](https://github.com/dimforge/nalgebra/pull/485/files#diff-425bf3710eefe907a4f8369b92cd4966) to the macro generating these methods:

<a href="https://docs.rs/nalgebra/0.21.1/nalgebra/base/type.Vector.html#impl-9"><iframe scrolling="no" style="pointer-events:none;height:90vh; width:100%" src="https://docs.rs/nalgebra/0.21.1/nalgebra/base/type.Vector.html#impl-9"></iframe></a>

**What changed?** The old macro generated an `impl` for each swizzling method:
```rust
impl<N: Scalar, D: DimName, S: Storage<N, D>> Vector<N, D, S> {
    pub fn xx(&self) -> Vector2<N>
    where
        D::Value: Cmp<typenum::U0, Output=Greater>
    { ... }
}

impl<N: Scalar, D: DimName, S: Storage<N, D>> Vector<N, D, S> {
    pub fn xxx(&self) -> Vector3<N>
    where
        D::Value: Cmp<typenum::U0, Output=Greater>
    { ... }
}

impl<N: Scalar, D: DimName, S: Storage<N, D>> Vector<N, D, S> {
    pub fn xy(&self) -> Vector2<N>
    where
        D::Value: Cmp<typenum::U1, Output=Greater>
    { ... }
}

/* and so on */
```

The new macro groups the methods into one of just three `impl`s depending on their dimensionality requirements:
```rust
// Swizzling methods for Vectors of dimension > 0
impl<N: Scalar, D: DimName, S: Storage<N, D>> Vector<N, D, S>
where
    D::Value: Cmp<typenum::U0, Output=Greater>
{
    pub fn xx(&self) -> Vector2<N>
    where
        D::Value: Cmp<typenum::U0, Output=Greater>
    { ... }

    pub fn xxx(&self) -> Vector3<N>
    where
        D::Value: Cmp<typenum::U0, Output=Greater>
    { ... }
}

// Swizzling methods for Vectors of dimension > 1
impl<N: Scalar, D: DimName, S: Storage<N, D>> Vector<N, D, S>
where
    D::Value: Cmp<typenum::U1, Output=Greater>
{
    pub fn xy(&self) -> Vector2<N>
    { ... }

    /* and so on */
}

// Swizzling methods for Vectors of dimension > 2
impl<N: Scalar, D: DimName, S: Storage<N, D>> Vector<N, D, S>
where
    D::Value: Cmp<typenum::U2, Output=Greater>
{
    pub fn xz(&self) -> Vector2<N>
    { ... }

    /* and so on */
}
```
...and rustdoc faithfully adheres to this organization when generating nalgebra's documentation!

**If you are generating `impl`s via a macro, check if your macro could be tweaked to group similar methods into the same `impl`!**

## Documenting `impl`s
**TIP: You can write documentations on individual `impl`s!**

Like Rust's slices, nalgebra's arrays allow for overloaded indexing; e.g.:
```rust
let matrix = Matrix3::new(0, 3, 6,
                          1, 4, 7,
                          2, 5, 8);

// index a particular element
assert_eq!(matrix.index((0, 0)), &0);

// select a range of rows and all columns
assert!(matrix.index((1..3, ..))
    .eq(&Matrix2x3::new(1, 4, 7,
                        2, 5, 8)));
```
...and these overloaded index types are usable with a whole suite of associated methods: `index`, `index_mut`, `get`, `get_mut`, `get_unchecked` and `get_unchecked_mut`. The same indexing types can be used on each of these methods—they only differ in their fallibility, mutability, and safety.

These six methods are grouped into the same `impl`. The documentation for the individual methods focuses just on their differences. Their similarities (namely, the different kinds of indexes which can be used) are documented *on this shared `impl`*:

<a style="display:block" href="https://docs.rs/nalgebra/0.22.0/nalgebra/base/struct.Matrix.html#impl-105"><iframe scrolling="no" style="pointer-events:none; height:90vh; width:100%" src="https://docs.rs/nalgebra/0.22.0/nalgebra/base/struct.Matrix.html#impl-105"></iframe></a>
 
**If you have thematically similar methods, you can group them into their own `impl`, and write rustdoc on that `impl`!**

Concretely:
```rust
/// # Indexing Operations
/// [documentation about indexing as a whole]
impl<N: Scalar, R: Dim, C: Dim, S: Storage<N, R, C>> Matrix<N, R, C, S> {
    /// [documentation *just* for `get`]
    #[inline]
    pub fn get<'a, I>(&'a self, index: I) -> Option<I::Output>
    where
        I: MatrixIndex<'a, N, R, C, S>,
    { ... }

    /// [documentation *just* for `get_mut`]
    #[inline]
    pub fn get_mut<'a, I>(&'a self, index: I) -> Option<I::Output>
    where
        S: StorageMut<N, R, C>,
        I: MatrixIndexMut<'a, N, R, C, S>,
    { ... }

    /* and so on */
}
