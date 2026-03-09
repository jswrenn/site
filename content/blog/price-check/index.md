+++
title = "Price-Checking Zerocopy's Zero Cost Abstractions"
date = 2026-03-09
tags = ["Rust", "zerocopy"]
+++

[Zerocopy](https://docs.rs/zerocopy/0.8.42/zerocopy/) is toolkit that promises safe and efficient abstractions for low-level memory manipulation and casting. While we've long done the usual (e.g., testing, documentation, abstraction, miri) and unusual (e.g., proofs, formal verification) to prove to ourselves and our users that we've kept our promise of safety, we've kept our other promise of efficiency with a less convincing combination of `#[inline(always)]` and faith in LLVM.

<!-- more -->

These two promises have increasingly been at odds. As the types and transformations supported by zerocopy have grown more complex, so too have our internal abstractions. Click through our docs into the source code of most of our methods and you will rarely see any *immediate* occurances of `unsafe`; we keep the dangerous stuff sequestered away a few function calls down in tightly-scoped "zero cost" abstractions. But *are* these abstractions actually zero cost?

Well, as of [zerocopy 0.8.42](https://docs.rs/zerocopy/0.8.42/zerocopy) trusting the optimizer requires a little less blind faith. We've begun documenting the codegen you can expect from each of zerocopy's routines in a representative range of circumstances; e.g., for [`FromBytes::ref_from_prefix`](https://docs.rs/zerocopy/latest/zerocopy/trait.FromBytes.html#method.ref_from_prefix.codegen):

<a href="https://docs.rs/zerocopy/latest/zerocopy/trait.FromBytes.html#method.ref_from_prefix.codegen"><img src="/blog/price-check/ref_from_prefix_docs.png" style="outline: 1px black solid" height="auto" width="100%" alt="Screenshot of codegen documentation for ref_from_prefix."></img></a>

This documentation surfaces the latest addition to our CI pipeline: code generation *testing*. The [`benches` directory](https://github.com/google/zerocopy/tree/main/benches) our our repo with a comprehensive set of microbenchmarks. Rather than actually executing these benchmarks on hardware, we use [`cargo-show-asm`](https://crates.io/crates/cargo-show-asm) to assert that their machine code and analysis matches model outputs checked into our repo. Consequently, we're able to verify our assumptions about how Rust and LLVM optimize our abstractions, and easily observe how our changes impact codegen.
