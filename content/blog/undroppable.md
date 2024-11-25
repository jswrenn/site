+++
title = "Undroppable Types"
date = 2024-11-25
tags = ["rust"]
+++

In Rust, `ManuallyDrop` is the tool of choice for eliding the destructor of a value. Wrapped in `ManuallyDrop`, a value is simply forgotten when it goes out of scope. However, in some cases, it is useful to ensure that a value *cannot* be dropped — not because it is forgotten, but because the very act of dropping it is a compilation error.

<!-- more -->

To achieve this, we only need to panic in a `const` context in its destructor; e.g.:

```rust
/// A type that cannot be dropped.
pub struct Undroppable<T: ?Sized>(mem::ManuallyDrop<T>);

impl<T> Undroppable<T> {
    // Makes `val` undroppable.
    //
    // If `val` has a  non-trivial destructor, attempting
    // to drop it will result in a compilation error.
    pub fn new_unchecked(val: T) -> Self {
        Self(mem::ManuallyDrop::new(val))
    }
}

impl<T:? Sized> Drop for Undroppable<T> {
    fn drop(&mut self) {
        const {
            assert!(!mem::needs_drop::<T>(), "This cannot be dropped.");
        }
    }
}
```

Unless we `mem::forget` a val wrapped in `Undroppable` (or further wrap it in `ManuallyDrop`); e.g.:

```rust
fn main() {
    let undroppable = Undroppable::new_unchecked(vec![1, 2, 3]);
    // commenting out this line results in a compilation error:
    core::mem::forget(undroppable);
}
```

...dropping it results in a compilation error:

```
error[E0080]: evaluation of `<Undroppable<std::vec::Vec<i32>> as std::ops::Drop>::drop::{constant#0}` failed
  --> src/main.rs:28:13
   |
28 |             assert!(!mem::needs_drop::<T>(), "This cannot be dropped.");
   |             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ the evaluated program panicked at 'This cannot be dropped.', src/main.rs:28:13
   |
   = note: this error originates in the macro `$crate::panic::panic_2021` which comes from the expansion of the macro `assert` (in Nightly builds, run with -Z macro-backtrace for more info)

note: erroneous constant encountered
  --> src/main.rs:27:9
   |
27 | /         const {
28 | |             assert!(!mem::needs_drop::<T>(), "This cannot be dropped.");
29 | |         }
   | |_________^

note: the above error was encountered while instantiating `fn <Undroppable<std::vec::Vec<i32>> as std::ops::Drop>::drop`
   --> /playground/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/core/src/ptr/mod.rs:574:1
    |
574 | pub unsafe fn drop_in_place<T: ?Sized>(to_drop: *mut T) {
    | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

This error message is poor as the 'stack' trace only extends one frame above our custom `drop` impl. However, in some scenarios, even a poor compilation error is preferable to a runtime panic. Hopefully, this diagnostic will be improved in future versions of rustc.

In [zerocopy](https://docs.rs/zerocopy/latest/zerocopy/), we may soon be using this approach eliminate a stubborn `T: Sized` bound from our `Unalign<T>` layout gadget, while ensuring that `Unalign` does not silently forget DSTs with non-trivial `Drop` implementations. 
