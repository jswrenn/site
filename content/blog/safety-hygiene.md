+++
title = "The Three Basic Rules of Safety Hygiene"
date = 2025-02-06
tags = ["rust"]
+++

> *Ask not what you can do with `unsafe`; ask what `unsafe` can do for you.*

On the Rust subreddit, I often see beginners get tripped up on `unsafe`, posing questions like *"What can I do in `unsafe`?"* and *"When do I need `unsafe`?"*. These are useful questions — but not enough to really understand (un)safety. The former defies a simple, evergreen answer — as Rust's operational semantics are refined, so too are what you can and cannot achieve with `unsafe` code. The latter begets teleological answers; you *need* unsafe when the sum of Rust's tooling requires you use `unsafe`. These answers are not particularly helpful for understanding how to write maintainable unsafe code, nor for informing how Rust's safety tooling might be improved. They miss the forest for the trees; they mistake the implementation for the idiom.

Let's take a step back. Just as Rust's borrow checker is the tooling that supports the idiom of ownership and *aliasing xor mutability*, the sum of Rust's safety tooling — a basket of keywords, well-formedness checks, and lints — is, too, the expression of a deeper idiom. In this essay, I give this idiom a name — *safety hygiene* — and argue that it's underpinned by three basic principles. In doing so, we gain a lens for understanding Rust's myriad safety tools, opportunities for improvement, and its greatest flamewars.

<!-- more -->

## An Introduction to Health and Safety

*Safety hygiene* is the practice of denoting and documenting where memory safety obligations arise and where they are discharged. The practice is underpinned by three principles:
1. Safety obligations are introduced with a key word and an explanation of the obligation.
2. Safety obligations are discharged with a key word and an explanation of why the obligation is discharged.
3. The key word and safety comments are avoided when no safety obligations are involved.

In Rust, that key word is `unsafe`, and the explanations of introduction and discharge are, conventionally, denoted with `# Safety` and `// SAFETY`. Its associated tooling is, basically, an implementation of these three rules.

### Function Safety Hygiene

For example, *function safety hygiene* is the practice of denoting and documenting when functions have safety conditions and when invocations discharge those conditions. A crate practicing good function safety hygiene will:
1. Denote such functions with the `unsafe` keyword and provide accompanying safety documentation.
2. Invoke such functions with the `unsafe` keyword and provide an accompanying safety comment.
3. Use `unsafe` narrowly in both cases. A function without safety preconditions shouldn't be marked `unsafe`, and `unsafe` should not be used to invoke safe functions.

Once the programmer makes the decision to mark a function `unsafe`, Rust's tooling provides support for each of the other elements of safety hygiene; respectively:
1. Clippy's [`missing_safety_doc`] lint ensures that `unsafe` functions have accompanying safety documentation.
2. Rust's well-formedness checker ensures (via [`E0133`]) that unsafe functions are invoked with the `unsafe` keyword. Clippy's [`undocumented_unsafe_blocks`] lint ensures that the invocation has accompanying safety documentation.
3. Rust's `unused_unsafe` lint ensures that `unsafe` is not used for completely safe operations.

[`missing_safety_doc`]: https://rust-lang.github.io/rust-clippy/master/index.html#missing_safety_doc
[`E0133`]: https://doc.rust-lang.org/stable/error_codes/E0133.html
[`undocumented_unsafe_blocks`]: https://rust-lang.github.io/rust-clippy/stable/index.html#undocumented_unsafe_blocks

### Trait Safety Hygiene

Similarly, *trait safety hygiene* is the practice of denoting and documenting when traits carry safety invariants, and when implementations satisfy those invariants. A crate practicing good trait safety hygiene will:
1. Denote such traits with the `unsafe` keyword and provide accompanying safety documentation.
2. Implement such traits with the `unsafe` keyword and provide an accompanying safety comment.
3. Use `unsafe` narrowly in both cases. A trait without safety conditions shouldn't be marked `unsafe`, and `unsafe` should not be used to implement safe traits.


Again, once the programmer makes the decision to mark a trait `unsafe`, Rust's tooling provides support for each of the other elements of safety hygiene; respectively:
1. Clippy's [`missing_safety_doc`] lint ensures that `unsafe` traits have accompanying safety documentation.
2. Rust's well-formedness checker ensures (via [`E0200`]) that unsafe functions are invoked with the `unsafe` keyword. Clippy's [`undocumented_unsafe_blocks`] lint ensures that the implementation has accompanying safety documentation.
3. Rust's well-formedness checker ensures (via [`E0199`]) that safe traits are not implemented with `unsafe`.

[`E0200`]: https://doc.rust-lang.org/stable/error_codes/E0200.html
[`E0199`]: https://doc.rust-lang.org/stable/error_codes/E0199.html

## The Implementation is Not the Idiom

Not only can these three principles be used to make sense of the idiom encouraged by Rust's tooling, but they also can inform how we can practice safety hygiene *beyond* the present limits of Rust's tooling.

### Field Safety Hygiene

In the [`zerocopy`](https://crates.io/crates/zerocopy) and [`itertools`](https://crates.io/crates/itertools) crates, we practice *field safety hygiene* — i.e., we denote and comment when fields carry safety invariants, and how uses of those field discharge those invariants.

For example, `zerocopy` defines a `Ptr<T, I>` type that wraps `NonNull<T>` with associated invariants `I`; e.g., whether reference is shared or exclusive, whether the referent is well-aligned, or a bit-valid instance of `T`. We enumerate each of these field invariants at the field definition site:

```rust
pub struct Ptr<'a, T, I>
where
    T: 'a + ?Sized,
    I: Invariants,
{
    /// # Invariants
    ///
    /// 0. If `ptr`'s referent is not zero sized, then `ptr` is derived from
    ///    some valid Rust allocation, `A`.
    /// 1. If `ptr`'s referent is not zero sized, then `ptr` has valid
    ///    provenance for `A`.
    /// 2. If `ptr`'s referent is not zero sized, then `ptr` addresses a
    ///    byte range which is entirely contained in `A`.
    /// 3. `ptr` addresses a byte range whose length fits in an `isize`.
    /// 4. `ptr` addresses a byte range which does not wrap around the
    ///     address space.
    /// 5. If `ptr`'s referent is not zero sized,`A` is guaranteed to live
    ///    for at least `'a`.
    /// 6. `T: 'a`.
    /// 7. `ptr` conforms to the aliasing invariant of
    ///    [`I::Aliasing`](invariant::Aliasing).
    /// 8. `ptr` conforms to the alignment invariant of
    ///    [`I::Alignment`](invariant::Alignment).
    /// 9. `ptr` conforms to the validity invariant of
    ///    [`I::Validity`](invariant::Validity).
    // SAFETY: `NonNull<T>` is covariant over `T` [1].
    //
    // [1]: https://doc.rust-lang.org/std/ptr/struct.NonNull.html
    ptr: NonNull<T>,
    // SAFETY: `&'a ()` is covariant over `'a` [1].
    //
    // [1]: https://doc.rust-lang.org/reference/subtyping.html#variance
    _invariants: PhantomData<&'a I>,
}
```
...and write `SAFETY` comments whenever `ptr` is initialized; e.g.:
```rust
impl<'a, T, I> Ptr<'a, T, I>
where
    T: 'a + ?Sized,
    I: Invariants,
{
    /// Constructs a `Ptr` from a [`NonNull`].
    ///
    /// # Safety
    ///
    /// The caller promises that:
    ///
    /// 0. If `ptr`'s referent is not zero sized, then `ptr` is derived from
    ///    some valid Rust allocation, `A`.
    /// 1. If `ptr`'s referent is not zero sized, then `ptr` has valid
    ///    provenance for `A`.
    /// 2. If `ptr`'s referent is not zero sized, then `ptr` addresses a
    ///    byte range which is entirely contained in `A`.
    /// 3. `ptr` addresses a byte range whose length fits in an `isize`.
    /// 4. `ptr` addresses a byte range which does not wrap around the
    ///    address space.
    /// 5. If `ptr`'s referent is not zero sized, then `A` is guaranteed to
    ///    live for at least `'a`.
    /// 6. `ptr` conforms to the aliasing invariant of
    ///    [`I::Aliasing`](invariant::Aliasing).
    /// 7. `ptr` conforms to the alignment invariant of
    ///    [`I::Alignment`](invariant::Alignment).
    /// 8. `ptr` conforms to the validity invariant of
    ///    [`I::Validity`](invariant::Validity).
    pub(super) unsafe fn new(ptr: NonNull<T>) -> Ptr<'a, T, I> {
        // SAFETY: The caller has promised to satisfy all safety invariants
        // of `Ptr`.
        Self { ptr, _invariants: PhantomData }
    }
}
```

By adhering strictly to this pattern throughout our codebase, we've been able to sustainably increase `zerocopy`'s scope without adding undue maintenance burden. It continues to be easy for us to extend unsafe abstractions weeks, months, or years after we last thought about them.

However, you might have noticed we have not *quite* adhered completely to the field safety principles in this example. Not only does Rust *not* provide tooling support for field safety hygiene, it actively works against us:
1. It is a parse error to apply the `unsafe` keyword to fields. Consequently, Clippy cannot prompt us to write a safety comment (but we remember to do so).
2. Rust's `unused_unsafe` lint discourages wrapping a 'safe' operation like `Self { ptr, _invariants: PhantomData }` in an `unsafe` block (and so we don't do so). Consequently, Clippy cannot prompt us to write a safety comment (but we remember to do so).


To a point, we can accommodate for these shortcomings by leveraging Rust's tooling for function safety hygiene. For example, if we are diligent about using `Ptr::new` instead of `Ptr { ... }`, Rust's function safety tooling *will* enforce the use of `unsafe` for construction. We can even nudge ourselves towards compliance by quarantining `Ptr`'s definition in its own module:

```rust
// Used to gate access to `Ptr`'s implicit constructor and fields.
mod def {
    pub struct Ptr<'a, T, I>
    where
        T: 'a + ?Sized,
        I: Invariants,
    {
        /// [Documentation elided.]
        ptr: NonNull<T>,
        // [Documentation elided.]
        _invariants: PhantomData<&'a I>,
    }
    
    impl<'a, T, I> Ptr<'a, T, I>
    where
        T: 'a + ?Sized,
        I: Invariants,
    {
        /// [Documentation elided.]
        pub(super) unsafe fn new(ptr: NonNull<T>) -> Ptr<'a, T, I> { … }
        
        /// [Documentation elided.]
        pub(crate) fn as_non_null(&self) -> NonNull<T> { … }
    }
}
```

In doing so, `Ptr`'s implicit constructors and fields are inaccessible outside of `def`; all construction and access is mediated through `Ptr::new` and `Ptr::as_non_null`.

### ...But the Implementation Defines the Convention

As these examples illustrate, going beyond Rust's safety hygiene tooling is fraught with linguistic friction. The implementation might not be the idiom, but it does set convention. I have not seen the `def` module pattern elsewhere — even within the standard library — and the practice of documenting field safety invariants is *much* rarer than the practice of documenting function safety invariants.

## The Implementation is Unfinished

Since 1.0, there have been countless proposals for improving Rust's safety tooling. Safety hygiene and its three rules gives us a framework for making sense of how these proposals refine tooling support for the idiom.

### Distinguishing Definition and Discharge

One class of proposals refines (un)safety by distinguishing the definition of safety conditions from the discharge of safety conditions. Presently, the same keyword is used for both case — `unsafe` — but some have argued that this overloads the keyword in a confusing way. [This 2024 pre-RFC](https://internals.rust-lang.org/t/split-keyword-unsafe-into-unsafe-checked/20334) proposes using `unsafe` to declare obligations, and `checked` to discharge them.


### Distinguishing Pre-Conditions and Post-Conditions

Another class of proposals refines (un)safety by distinguishing pre-conditions from post-conditions. Presently the `unsafe` keyword and Safety documentation does double-duty in both being used to denote and document when uses must uphold safety preconditions, and denoting and documenting when uses can rely on safety-critical postconditions. [This pre-RFC from 2018](https://internals.rust-lang.org/t/pre-rfc-another-take-at-clarifying-unsafe-semantics/7041), for example, proposes using `unsafe(pre)` for the former case and `unsafe(post)` to denote the latter.

### Structured Safety Documentation

The last class of proposals we'll consider refines (un)safety by adding tooling for checking that the correspondence between definition and discharge is (correctly) documented. This [2017 RFC](https://github.com/rust-lang/rfcs/pull/1910) proposed an attribute alternative to the status quo of unstructured safety comments. This [2024 pre-RFC](https://internals.rust-lang.org/t/pre-rfc-unsafe-reasons/22093) proposes an extension to `unsafe` where definition-site uses of `unsafe` enumerate labeled safety conditions, and use-site `unsafe` must discharge those obligations by enumerating the same labels.


## Next Steps

Besides the aforementioned proposals to extend the sophistication of Rust's safety hygiene tooling, there are also more basic steps we can to improve the consistency and utility of the present safety tooling.

### Opinionated Linting Defaults

A funky identifier, or undocumented unsafe code — what's worse? Presently, Rust does more to discourage the former than the latter. Whereas the `non_upper_case_globals` and `non_camel_case_types` live in rustc and are executed, by default, on every build, the `missing_safety_doc` and `undocumented_unsafe_blocks` lints are relegated to Clippy. The `undocumented_unsafe_blocks` lint is even allow-by-default!

If we believe that good safety hygiene is a core Rust idiom — just as its naming conventions are — then these safety hygiene lints should be migrated to rustc and warn-by-default.

### Tooling for Field Safety Hygiene

[RFC3458](https://github.com/rust-lang/rfcs/pull/3458) (of which I am a co-author) proposes extending Rust's safety hygiene tooling to fields. In brief, it would encourage the use of `unsafe` to denote fields with safety invariants, and require the use of `unsafe` to discharge those safety invariants. The proposal uses the three principles of safety hygiene as its starting design tenets, and follows them to their conclusion to inform the design.


### Safe Unions

We can also use this framework to make sense of a language design corner we have seemingly painted ourselves into: safer unions. Unions, in Rust, may presently be safely defined, constructed and mutated — but require `unsafe` to read. Consequently, it is possible to place an union into a state where its fields cannot be soundly read, using only safe code:

```rust
#[derive(Copy, Clone)] #[repr(u8)] enum Zero { V = 0 }
#[derive(Copy, Clone)] #[repr(u8)] enum One  { V = 1 }

union Tricky {
    a: (Zero, One),
    b: (One, Zero),
}

let mut tricky = Tricky { a: (Zero::V, One::V) };
tricky.b.0 = One::V;

// Now, neither `tricky.a` nor `tricky.b` is in a valid state!
```

The possibility of such unions makes it tricky to retrofit a mechanism for safe access. Because `unsafe` was not required to define or mutate this `union`, the invariant that makes reading sound is entirely implicit. The Rust compiler has no way of knowing if or how a union's user intends to uphold this invariant, and thus its safety tooling can only assume that unions are *never* in a valid state.

Unsafe unions are undeniably useful, but the fact that all unions are *implicitly* unsafe breaks from Rust's usual convention that items are safe-by-default and takes up valuable syntactic space that could instead be occupied by *safe* unions.

Rust could free up this syntactic space by migrating its implicitly unsafe unions to be explicitly unsafe, across an edition boundary; e.g.:
```rust
union MaybeUninit<T> {
    uninit: (),
    unsafe value: ManuallyDrop<T>,
}
```

## Parting Thoughts

I have more to say on this. I believe the lens of safety hygiene can help make sense of the 2018 Actix drama, but with so many links rotted, explaining this requires more archeology than would flow neatly in this essay. The implementation might not be the idiom, but the gaps in Rust's implementation of safety tooling fuel disagreements over what, exactly, the idiom is. By filling these gaps, we can work towards a future where Rust's safety story *really* is just these three rules, and its safety tooling can be explained orthogonally from Rust's other abstraction mechanisms. Stay tuned!
