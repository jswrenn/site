+++
title = "Oh the Crates You'll Go! A 2022 Retrospective"
date = 2022-01-04
tags = ["Rust"]
+++

As my first full calendar year in the Rust Platform Team at AWS draws to a close, I thought it might be illuminating to reflect on what I worked on. The breadth of crates I released or contributed to surprised me! Time really does fly when you're having fun.

<!-- more -->

Throughout the year, I chatted with programmers building networked services in Rust about their pain-points. The driving theme of those conversations was *observability*. Rust is not so inherently fast that good architecture is irrelevant. When performance isn't satisfactory, discovering why can be harder than it is in some 'slower' languages like Javascript or Java. Rust's speed comes at a cost: It leaves it up to *you* to carefully instrument your program. I spent my year building the tools to help you do this.

So, without further ado, here's a rundown of the crates I released or significantly contributed to in 2022:
 - [`tokio-metrics`](#tokio-metrics)
 - [`tracing-allocations`](#tracing-allocations)
 - [`type-census`](#type-census)
 - [`tracing`](#interlude-tracing)
 - [`fn_name`](#fn-name)
 - [`#![feature(transmutability)]`](#feature-transmutability)
 - [`tracing-causality`](#tracing-causality)
 - [`async-backtrace`](#async-backtrace)
 - [`deflect`](#deflect)


## `tokio-metrics`
Get insight into the performance of async tasks and the tokio runtime. Key links:
- [announcement](https://tokio.rs/blog/2022-02-announcing-tokio-metrics)
- [crates.io](https://crates.io/crates/tokio-metrics)
- [docs.rs](https://docs.rs/tokio-metrics/0.1.0/tokio_metrics/)
- [github.com](https://github.com/tokio-rs/tokio-metrics)

How much time is a task spending scheduled, running, or waiting? Am I hitting bottlenecks in the Tokio runtime itself? The `tokio-metrics` crate provides tools to help answer these questions, and more. At the start of 2022, I picked up development of the tokio-metrics crate from [Carl Lerche](https://carllerche.com/) and drove the prototype to its initial release.

To monitor metrics of one or more async tasks with this crate, you construct a `TaskMonitor`, and `.instrument` your tasks with it before spawning them. For example:
```rust
// construct a TaskMonitor
let monitor = tokio_metrics::TaskMonitor::new();

// print task metrics every 500ms
{
    let frequency = std::time::Duration::from_millis(500);
    let monitor = monitor.clone();
    tokio::spawn(async move {
        for metrics in monitor.intervals() {
            println!("{:?}", metrics);
            tokio::time::sleep(frequency).await;
        }
    });
}

// instrument some tasks and spawn them
loop {
    tokio::spawn(monitor.instrument(do_work()));
}
```
The `TaskMonitor` abstraction lets you be as granular with metrics collection as you'd like. A single `TaskMonitor` can be shared for, say, all tasks spawned at a particular source code location, or (after narrowing in on a performance issue) a distinct `TaskMonitor` can be used for distinct tasks. The [documentation for `TaskMonitor`](https://docs.rs/tokio-metrics/0.1.*/tokio_metrics/struct.TaskMonitor.html#why-are-my-tasks-slow) walks through how one might use the crate to answer *"Why are my tasks slow?"*.

Task monitoring is only half the crate. The other half, `RuntimeMonitor`, provides a similar API for getting metrics from the tokio runtime. Although the crate is called *tokio*-metrics, the `TaskMonitor` machinery of the crate *doesn't* depend on tokio --- you can use it to monitor tasks scheduled with `async-std` and `smol`, too.

Looking back on this crate, I'm particularly proud of the documentation I wrote --- lines of documentation exceed lines of code by more than a factor of three, and every metric has a doctest illustrating a scenario that would impact that metric.

## `tracing-allocations`
A [global allocator](https://doc.rust-lang.org/stable/std/alloc/trait.GlobalAlloc.html) that emits [tracing](https://crates.io/crates/tracing) events upon allocation and deallocation. Key links:
- [crates.io](https://crates.io/crates/tracing-allocations)
- [docs.rs](https://docs.rs/tracing-allocations)
- [github.com](https://github.com/jswrenn/tracing-allocations)

Where is my asynchronous application allocating and deallocating memory? That is the question I attempted to answer with the `tracing-allocations` crate. The crate provides a `TracingAllocator`, which wraps any global allocator and augments it with tracing events. To use the crate, one simply writes:
```rust
use std::alloc::System;
use tracing_allocations::TracingAllocator;

#[global_allocator]
static GLOBAL: TracingAllocator<System> = TracingAllocator::new(System);
```
Then---in theory---allocation events will appear seamlessly in whatever tool you're using to analyze tracing logs.

Although the crate works as advertised, I found that it was impractical to completely control for reentrant allocations; i.e., ensuring that whatever tracing layers you installed to process your application's tracing events did not, themselves, allocate memory when processing the events emitted by the tracing allocator. You might, for instance, want to *print* allocation events to stdout. Tough luck! The `println!` macro, under-the-hood, *buffers* output to stdout, and might lock and (re)allocate that buffer in the process of printing. Thus, if you try to print *while* this buffer is being (re)allocated, your application will deadlock.

While it's possible to workaround this particular footgun, I could not find a way to mitigate *all* footguns of this genre. I cannot advise using this crate unless your tracing layers anticipate it, and make a point of turning off allocation tracing at any points they allocate. The crate stands as an imperfect experiment.

## `type-census`
Audit the population counts of types. Key links:
- [crates.io](https://crates.io/crates/type-census)
- [docs.rs](https://docs.rs/type-census)
- [github.com](https://github.com/jswrenn/type-census)

In the aftermath of `tracing-allocations`, I circled back service teams and learned that one question they'd really like to answer, at the very least, was *"How many instances of a given type are there?"*. I developed the `type-census` crate to provide an efficient mechanism for answering this question. It's used like so:
```rust
// 1. import these two items:
use type_census::{Instance, Tabulate};

// 2. Derive `Tabulate`
#[derive(Clone, Tabulate)]
pub struct Foo<T> {
    v: T,
    // 3. add a field of type `Instance<Self>`
    _instance: Instance<Self>,
}

impl<T> Foo<T> {
    pub fn new(v: T) -> Self
    where
        // 4. add a `Self: Tabulate` bound to constructors
        Self: Tabulate,
    {
        Self {
            v,
            // 5. and initialize your `Instance` field like so:
            _instance: Instance::new(),
        }
    }
}
```

Then, you can call `T::instances()` to get the population count of `T`; e.g.:
```rust
assert_eq!(Foo::<i8>::instances(), 0);

let mut bar: Vec<Foo<i8>> = vec![Foo::new(0i8); 10];

assert_eq!(Foo::<i8>::instances(), 10);

let _ = bar.drain(0..5);

assert_eq!(Foo::<i8>::instances(), 5);
```

The crate has its limitations (or, depending on your use-case, features). The same instance counter is shared between all instantiations of generic types; i.e., `Foo<u8>` and `Foo<i8>` would both report a population count of 5 at the end of the above example. It also requires explicit code modifications---a far cry from the convenience of a Java heap dump.

## Interlude: `tracing`
The [`tracing`](https://docs.rs/tracing) family of crates brings structured logging to Rust. I contributed to `tracing` significantly in 2022:
- [permit setting parent span via `#[instrument(parent = …)]`](https://github.com/tokio-rs/tracing/pull/2091)
- [permit `#[instrument(follows_from = …)]`](https://github.com/tokio-rs/tracing/pull/2093)
- [more `downcast_ref` & `is` methods](https://github.com/tokio-rs/tracing/pull/2160)
- [implement `Collect` for `Box<C>`, `Arc<C>`](https://github.com/tokio-rs/tracing/pull/2161)
- [implement `PartialEq`, `Eq` for `Metadata`, `FieldSet`](https://github.com/tokio-rs/tracing/pull/2229)
- [implement `LookupSpan` for `Box<LS>` and `Arc<LS>`](https://github.com/tokio-rs/tracing/pull/2247)
- [add `{Collect,Subscriber}::on_register_dispatch`](https://github.com/tokio-rs/tracing/pull/2269)
- [add `Dispatch::downgrade()` and `WeakDispatch`](https://github.com/tokio-rs/tracing/pull/2293)

Many of these changes were made to support my work on my own [`tracing-causality`](#tracing-causality) crate (more on that later)!

## `fn_name`
Macros that expand to the name of the function they're invoked within. Key links:
- [crates.io](https://crates.io/crates/fn_name)
- [docs.rs](https://docs.rs/fn_name)
- [github.com](https://github.com/jswrenn/fn_name)

I developed this after discovering that the `tracing` crate's `#[instrument]` attribute macro [*only* sees the name of the method it's applied to, not the name of the type the method is on](https://github.com/tokio-rs/tracing/issues/2116).

The crate provides two macros. The `fn_name::uninstantiated!()` macro produces the name of the surrounding function with generics left uninstantiated; e.g.:
```rust
struct GenericType<A>(A);

impl<A> GenericType<A> {
    fn generic_method<B>(self, _: B) -> &'static str {
        fn_name::uninstantiated!()
    }
}

assert_eq!(
    GenericType(42u8).generic_method(false),
    "GenericType<_>::generic_method"
);
```
The `fn_name:instantiated!()` macro produces the name of the surrounding function, including generic types:
```rust
struct GenericType<A>(A);

impl<A> GenericType<A> {
    fn generic_method<B>(self, _: B) -> &'static str {
        fn_name::instantiated!()
    }
}

assert_eq!(
    GenericType(42u8).generic_method(false),
    "GenericType<u8>::generic_method<bool>"
);
```

Unfortunately, the crate is a little hacky under-the-hood; e.g.:
```rust
macro_rules! instantiated {
    () => {{
        fn type_name_of_val<T: ?Sized>(_: &T) -> &'static str {
            core::any::type_name::<T>()
        }
        const PREFIX: &str = concat!(module_path!(), "::");
        const SUFFIX: &str = "::{{closure}}";
        let here = &type_name_of_val(&||{});
        &here[PREFIX.len()..(here.len() - SUFFIX.len())]
    }}
}
```
These macros cannot be used in `const` contexts due to the dependency on [`type_name`](https://doc.rust-lang.org/std/any/fn.type_name.html), and it's probably not too appropriate to parse `type_name`'s output, anyways. I'd love to see official versions of them as part of Rust, proper!

## `#![feature(transmutability)]`
In 2021, I submitted [MCP411](https://github.com/rust-lang/compiler-team/issues/411), which proposes adding to Rust a compiler-supported analysis of when the bytes of one type can be safely reinterpreted as if they belong to another type. In 2022, I [began to implement it](https://github.com/rust-lang/rust/issues/99571)!

On nightly Rust, you can now use `#![feature(transmutability)]` to verify the transmutability of primitive and user-defined types with specified layouts:
```rust
#![feature(transmutability)]

#[repr(C)] struct Foo(u8, bool);
#[repr(C)] struct Bar(u16);

assert::is_transmutable::<Foo, Bar>(); // Compiler accepts this!
assert::is_transmutable::<Bar, Foo>(); // ERROR! 

mod assert {
    use std::mem::BikeshedIntrinsicFrom as TransmutableFrom;
    pub struct Context;

    pub fn is_transmutable<Src, Dst>()
    where
        Dst: TransmutableFrom<Src, Context>
    {}
}
```

What remains to be done: support for references, optimizations, and better error messages. Nonetheless, I'm extraordinarily happy that the end of my safe transmute saga---begun in 2019---is now within sight.

## `tracing-causality`
A crate for tracking the causal relationships between tracing spans. Key links:
- [crates.io](https://crates.io/crates/tracing-causality)
- [docs.rs](https://docs.rs/tracing-causality)
- [github.com](https://github.com/jswrenn/tracing-causality)

Although the `tracing` family of crates brings structured logging to Rust, they do not provide much out-of-the-box functionality for exploring those structures from within the application. This functionality is invaluable for instrumenting async applications. *What is an async task doing?* Just walk down its tree of tracing spans to find out!

This crate provides mechanisms for doing just that---it provides a tracing layer that tracks causal relationships, and [an interface for efficiently querying those relationships](https://docs.rs/tracing-causality/0.1.*/tracing_causality/fn.trace.html).

I used this crate to explore [adding tracing-driven 'stack' traces](https://github.com/tokio-rs/console/pull/363) to [Tokio Console](https://tokio.rs/blog/2021-12-announcing-tokio-console), but ultimately abandoned that work in favor of the [`async-backtrace`](#async-backtrace). I still believe that graphical tracing-driven stack traces are valuable, and welcome any readers to revive my draft PR to Tokio Console!

## `async-backtrace`
A crate for efficiently tracking and querying the call tree of `async` functions. Key links:
- [announcement](https://tokio.rs/blog/2022-10-announcing-async-backtrace)
- [crates.io](https://crates.io/crates/async-backtrace)
- [docs.rs](https://docs.rs/async-backtrace)
- [github.com](https://github.com/tokio-rs/async-backtrace)

In synchronous, multi-threaded applications, you can investigate deadlocks by inspecting stack traces of all running threads. Unfortunately, this approach breaks down for most asynchronous Rust applications, since suspended tasks — tasks that are not actively being polled — are invisible to traditional stack traces. The `async-backtrace` crate fills this gap, allowing you see the state of these hidden tasks.

To use it, annotate your `async` functions with `#[async_backtrace::framed]`; e.g.:
```rust
#[async_backtrace::framed]
async fn foo() {
    bar().await;
}

#[async_backtrace::framed]
async fn bar() {
    baz().await;
}

#[async_backtrace::framed]
async fn baz() {
    std::future::pending::<()>().await
}
```
...and call `taskdump_tree`:
```rust
#[tokio::main(flavor = "current_thread")]
async fn main() {
    tokio::select! {
        // run the following branches in order of their appearance
        biased;

        // spawn task #1
        _ = tokio::spawn(foo()) => { unreachable!() }

        // spawn task #2
        _ = tokio::spawn(foo()) => { unreachable!() }

        // print the running tasks
        _ = tokio::spawn(async {}) => {
            println!("{}", async_backtrace::taskdump_tree(true));
        }
    };
}
```
Running the above program prints out traces for each task:
```text
╼ foo::{{closure}} at example.rs:22:1
  └╼ multiple::bar::{{closure}} at example.rs:27:1
     └╼ multiple::baz::{{closure}} at example.rs:32:1
╼ foo::{{closure}} at example.rs:22:1
  └╼ bar::{{closure}} at example.rs:27:1
     └╼ baz::{{closure}} at example.rs:32:1
```

Under-the-hood, this crate does some *very* cool things: with minimal locking, it builds up an intrusive, doubly-linked tree of `Future`s as they are polled for the first time, and tears down the tree as they are dropped. It's the gnarliest `unsafe` code I've ever written, and I'm indebted to [miri](https://github.com/rust-lang/miri) and [loom](https://github.com/tokio-rs/loom) for helping me identify aliasing and concurrency bugs.

## `deflect`
Native reflection for Rust. Key links:
- [announcement](../deflect)
- [crates.io](https://crates.io/crates/deflect)
- [docs.rs](https://docs.rs/deflect)
- [github.com](https://github.com/jswrenn/deflect)

Rounding out the year, I released `deflect`, a library implementing reflection for Rust. It uses DWARF debuginfo to dynamically interpret types, much like a native debugger (e.g., GDB, LLDB, etc.). With it, you can:
- recover the concrete type of any trait object
- index-by-name or iterate-over the fields of a `struct`
- examine the captured data of a closure
- examine the internal structure of Rust `async fn` generators
- pretty-print arbitrary data (even if it doesn't implement `Debug`!)

Here is an example showing off a few of these capabilities:

```rust
// define a datatype
// note that it doesn't `derive` any traits
struct Foo {
    a: u8
}

// initialize the reflection info provider
let context = deflect::default_provider()?;

// create some type-erased data
let erased: Box<dyn Any> = Box::new(Foo { a: 42 });

// cast it to `&dyn Reflect`
let reflectable: &dyn deflect::Reflect = &erased;

// reflect it!
let value: deflect::Value = reflectable.reflect(&context)?;

// pretty-print the reflected value
// note that the concrete type `Foo` has been recovered from `dyn Any`!
assert_eq!(value.to_string(), "box Foo { a: 42 }");
 
// downcast into a `BoxedDyn` value
let value: deflect::value::BoxedDyn = value.try_into()?;

// dereference the boxed value
let value: deflect::Value = value.deref()?;
 
// downcast into a `Struct` value
let value: deflect::value::Struct = value.try_into()?;
 
// get the field `a` by name
let Some(field) = value.field("a")? else {
    panic!("no field named `a`!")
};
 
// get the value of the field
let value: deflect::Value = field.value()?;
 
// downcast into a `u8`
let value: u8 = value.try_into()?;
 
// check that it's equal to `42`!
assert_eq!(value, 42);
```

While I won't regurgitate my [entire announcement post for `deflect`](../deflect) (go read it!), I will reiterate the story of why my year ended here. In the wake of publishing `async-backtrace`, I found that few people relish the thought of annotating *every* `async` fn in their code base. It was around this time that I learned that [Tyler Mandry][tmandry] was [experimenting with DWARF-directed async backtraces][zulip], capable of providing traces of arbitrary `async fn`s. His proof of concept was compelling, but generalizing it to *arbitrary* `Future`s (particularly, manually-implemented ones) would require the ability to reflect *arbitrary* types. Thus, deflect was born.

[tmandry]: https://github.com/tmandry
[zulip]: https://rust-lang.zulipchat.com/#narrow/stream/187312-wg-async/topic/Async.20stack.20trace.20support.20library/near/306350471

## What's next?
I hope 2023 is as fruitful as 2022! I plan to continue talking to folks about their Rust pain-points, and doing what I can to resolve them. I could not have predicted last January the problems I would tackle in the year to come, so I'd be foolish to think I could predict everything 2023 holds for me.

Nonetheless, I *do* have some unfinished business. For one, rustc's transmutability analysis still lacks support for references; I've begun chipping away at that. And for `deflect`, I hope to implement pretty-printed stacktraces of arbitrary futures. Lastly, I have a few yet-to-be-explored ideas for how to improve the Tokio performance debugging story---stay tuned!

## Thanks
I'm tremendously grateful for the time, ideas and expertise of my colleagues, and of all those who shared with me their pain-points of developing networked services in Rust. And, I'm tremendously grateful to my employer for the freedom and opportunity to keep pushing the bounds of what’s possible in Rust. Little of what I mentioned in this blog post would have been possible without all of the aforementioned support.

## Personal Postscript
I had a lovely year non-professionally, too. I completed my PhD. I began to learn Spanish. My girlfriend and I embarked on our most ambitious bicycle tours, yet! We're getting a tandem bicycle this year---I can't wait to see where it takes us. My most professionally productive periods in 2022 were, paradoxically, the same periods in which I made the most time for myself, nature, and my loved ones. I'm hoping to approach 2023 with renewed intentionality: to seek out more moments for reflection, to tread more lightly on this earth, and to experience more of this world's wonders.
