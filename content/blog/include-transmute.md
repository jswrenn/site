+++
title = "Fixing include_bytes!"
date = 2020-08-26
tags = ["rust"]
+++

Rust's [`include_bytes!` macro](https://doc.rust-lang.org/core/macro.include_bytes.html) lets you statically include the contents of a file into your executable's binary. The builtin is a quick-and-dirty solution for packaging data with your executable, and perhaps even helping the compiler optimize your code! **Unfortunately, it's difficult to use correctly.**

<!-- more -->

A few years ago, for coursework, I implemented a basic neural network for recognizing digits. Ever-obsessed with having the *fastest* implementation in the class, I statically included the learned network parameters:
```rust
pub fn recognize(input: &Matrix<f64, U1, U784>) -> usize
{
    static RAW_WEIGHT : &'static [u8; 62_720] = include_bytes!("/weight.bin");

    static RAW_BIAS : &'static [u8; 80] = include_bytes!("/bias.bin");

    let WEIGHT: &Matrix<f64, U784, U10> = unsafe{ mem::transmute(RAW_WEIGHT) };

    let BIAS: &Matrix<f64, U1, U10> = unsafe{ mem::transmute(RAW_BIAS) };

    network::recognize(input, WEIGHT, BIAS)
}
```

**This is wrong, and I got lucky.**

A `Matrix<f64, U784, U10>` has a minimum alignment of `8`, but `include_bytes!` merely produces a reference to a byte array—with no care for alignment.

**How can we ensure the included bytes are properly aligned?**

## Prior Art: `include_bytes_align_as!`
On the Rust Internals Forum, user ExpHP [devised an `include_bytes_align_as` macro](https://users.rust-lang.org/t/can-i-conveniently-compile-bytes-into-a-rust-program-with-a-specific-alignment/24049/2) that produces the included bytes, aligned a given type:
```rust
macro_rules! include_bytes_align_as {
    ($align_ty:ty, $path:literal) => {{
        #[repr(C)]
        pub struct AlignedAs<Align, Bytes: ?Sized> {
            pub _align: [Align; 0],
            pub bytes: Bytes,
        }

        static ALIGNED: &AlignedAs::<$align_ty, [u8]> = &AlignedAs {
            _align: [],
            bytes: *include_bytes!($path),
        };

        &ALIGNED.bytes
    }};
}

#[repr(align(4096))]
struct Align4096;

static A: &'static [u8] = include_bytes!("boxed.rs");
static B: &'static [u8] = include_bytes_align_as!(f64, "boxed.rs"); // alignment of 8
static C: &'static [u8] = include_bytes_align_as!(Align4096, "boxed.rs"); // alignment of 4096
```
The implementation of this macro exploits a really neat trick: the type `AlignedAs` will have the alignment characteristics of `Align`, but contain the bytes of `Bytes`.

Unfortunately, Rust cannot infer the correct alignment for us—it's entirely up to pass the macro a type that exemplifies our desired alignment. Can we improve on this?

## An Alternative: `include_transmute!`
**Yes!** If we fuse the include and transmutation together, Rust will correctly-align its output *without* an extra macro argument; e.g.:
```rust
pub fn recognize(input: &Matrix<f64, U1, U784>) -> usize
{
    static WEIGHT: Matrix<f64, U784, U10> = unsafe{ include_transmute!("/weight.bin") };

    static BIAS: Matrix<f64, U1, U10> = unsafe{ include_transmute!("/weight.bin") };

    network::recognize(input, &WEIGHT, &BIAS)
}
```
Here's how:
```rust
macro_rules! include_transmute {
    ($file:expr) => {
        &core::mem::transmute(*include_bytes!($file))
    };
}
```
That's really all there is to it!

## Acknowledgements
Thanks to [/u/RustMeUP](https://www.reddit.com/user/RustMeUp) for [finding the bug](https://www.reddit.com/r/rust/comments/igi6p0/prerfc_safer_transmutation/g2wt27y/) in my motivating example!
