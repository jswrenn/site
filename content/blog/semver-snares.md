+++
title = "Rust's SemVer Snares: Introduction"
date = 2021-01-03
tags = ["rust","semver"]
+++

At the heart of [Semantic Versioning](https://semver.org/) is a distinction between incompatible API changes ("breaking" changes) and backwards-compatible API changes ("non-breaking" changes). When you make a change that could break existing code, you must increment the `MAJOR` component of its version number. Since failing to do this correctly may cause builds or tests to fail seemingly-spontaneously, knowing what kinds of changes are breaking and which are not is crucial to good crate hygiene.

<!-- more -->

Ideally, the affordances of a programming language make it obvious when a change can be made without breaking *well-behaving* downstream code. A comprehensive visibility system, for instance, goes a *long* way to achieving this ideal. Unfortunately, visibility perhaps goes *so* far towards this ideal, that it can be all-the-more easy to forget where it *fails* to communicate a breaking change.

In this series, I'll examine Rust's many SemVer *snares* — the subtle ways in which you might unexpectedly break downstream code or inadvertently expose an aspect of your API that you never intended to be stable. But first...

## Scope
Since SemVer is, above all, a social contract between programmers, this series will be deeply colored by my personal opinions about the social responsibilities of Rust programmers. Let's get acquainted!

### Responsibilities of Language Designers
Without a common understanding between library producers and consumers of what constitutes a breaking change, SemVer is practically worthless. Language designers (in the broadest sense of the term) are responsible for setting the ground-rules of API stability.

A language designer optimizing for SemVer stability would probably operate on the dual assumptions that:
1. library consumers will [rely](https://xkcd.com/1172/) on any programmatically observable qualities of a library
2. producers will need fine-grained control over the observable qualities of their libraries

...and thus develop a language with consistent, fine-grained controls over all observable qualities of a library, and cultivate a community understanding of "breaking change" that's tied solely to these controls.

**However, SemVer stability *isn't* the *sole* priority of language designers.** It's okay to have language features that *don't* connote API stability (like `#[repr(C)]`), to have intrinsics that make observable aspects of types that aren't API stable (like `mem::align_of`), and to provide mechanisms that achieve instability-through-obscurity (like `#[doc(hidden)]`) — so long as the stability implications of these mechanisms are well-documented.

This series will not, generally speaking, rail against the existence of these features, but it might point out instances where their caveats are poorly communicated.

### Responsibilities of Producers
My model crate author *tries their best* to not break consumers of their crates. They are well-informed (but not necessarily *perfectly* informed) about Rust's stability guidelines.

The model author is cautious about expanding the stable API surface of their crate. If they cannot make some unstable aspect of their crate unobservable, they obscure it (e.g., via `#[doc(hidden)]`). If they cannot obscure it, they clearly document the instability.

The model crate author is the target audience of this series. The entries will cover situations in which Rust's stability guidelines are unclear or inconsistent, and tactics for minimizing the stable API surface of a crate.

### Responsibilities of Consumers
My model crate consumer's use of a crate is guided by that crate's documentation and by Rust's error messages. For the sake of this series, I will *generally* assume that crate consumers do not go out of their way to rely on unspecified behavior. (And, if they do, they have pinned the exact version of the crate, and won't file a bug report when a `cargo update` breaks their build.)

## Snares
- [Sizedness and Size](/blog/semver-snares-size)
- [Alignment](/blog/semver-snares-alignment)
