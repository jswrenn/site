+++
title = "Announcing `scoped-trace`."
date = 2023-03-25
tags = ["Rust", "scoped-trace"]
+++

Today, I'm announcing `scoped-trace`, a crate for capturing scoped, tree-like execution traces.

Here are the crate's important links:
- [https://crates.io/crates/scoped-trace](https://crates.io/crates/scoped-trace)
- [https://docs.rs/scoped-trace](https://docs.rs/scoped-trace)
- [https://github.com/jswrenn/scoped-trace](https://github.com/jswrenn/scoped-trace)

<!-- more -->

For example, running this program:
```rust
use scoped_trace::Trace;

fn main() {
    // `Trace::root` establishes the upper unwinding bound of traces.
    let (_, trace) = Trace::root(|| foo());
    println!("{trace}");
}

fn foo() {
    bar();
    baz();
}

fn bar() {
    // `Trace::leaf` establishes the lower bounds of traces.
    Trace::leaf();
}

fn baz() {
    Trace::leaf();
}
```
...produces an output that looks like this:
```
╼ inlining::main::{{closure}} at example.rs:5:38
  ├╼ inlining::foo at example.rs:10:5
  │  └╼ inlining::bar at example.rs:16:5
  └╼ inlining::foo at example.rs:11:5
     └╼ inlining::baz at example.rs:20:5
```
This trace stops at the upper unwinding bound established by `Trace::root`, and includes the callers of `Trace::leaf`. The trace also preserves the order of execution, showing that `bar()` was invoked before `baz()`.

A quirk of this crate is that it inverts the usual directionality of *back*trace APIs. Calling `std::backtrace::Backtrace::capture()`, for instance, *gives* you a backtrace that starts at its callsite and unwinds to the top of the stack. By contrast, calling `Trace::leaf` gives you nothing at all; rather, it contributes itself to the trace produced by `Trace::root`. In a sense `Trace::root` doesn't give up a *back*trace, it gives you a *down*trace --- a view *down* the call stack starting at `Trace::root`.

## Backstory

This crate is one of a several of crates I developed recently to explore the solution space of tracing the state of asynchronous tasks. You can read more about that work [here](https://hackmd.io/@jswrenn/SkldX98Ci); this crate is a proof-of-concept of the "Backtrace the Leaves" approach described there.

In contrast to the [`async-backtrace`](https://tokio.rs/blog/2022-10-announcing-async-backtrace) crate, which requires onerous manual instrumentation and imposes a constant, modest runtime overhead, the `scoped-trace` approach will only require instrumenting `Future` leaves (i.e., futures without sub-futures, like timers or I/O), and does not impose any runtime overhead. I've begun experimenting integrating this approach into Tokio proper, where you'll be able to use it to inspect the state of all tasks managed by a Tokio runtime. Stay tuned!
