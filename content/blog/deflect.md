+++
title = "Native Reflection in Rust"
date = 2022-12-15
tags = ["Rust", "deflect"]
+++

Today, I'm releasing ***deflect***, an implementation of [reflection] for Rust. Deflect can be used to recover the concrete types of trait objects, inspect the internal state of `async` generators, pretty-print arbitrary data, and much more.

[reflection]: https://en.wikipedia.org/wiki/Reflection

<!-- more -->

Here are the crate's important links:
- [https://crates.io/crates/deflect](https://crates.io/crates/deflect)
- [https://docs.rs/deflect](https://docs.rs/deflect)
- [https://github.com/jswrenn/deflect](https://github.com/jswrenn/deflect)

## What?
Reflection is the ability of a program to inspect its own structure and behavior. In Javascript, for example, it's possible (and quite common) to write programs that iterate over the key--value pairs of arbitrary objects, or check that an object contains a field of a given name. Deflect brings some of these capabilities to Rust.

At the heart of deflect is its `Reflect` trait, which is implemented for all types. With it, you can:
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

## How?
Other notable implementations of reflection for Rust include [bevy_reflect] and [frunk]. These crates provide reflection traits that can be *explicitly* implemented for particular types, usually via a proc-macro `derive`. They are reliable, featureful crates that leverage and complement Rust's linguistic facilities. These crates are suitable for any use case where you know in advance which types you'll need to reflect.

[bevy_reflect]: https://crates.io/crates/bevy_reflect
[frunk]: https://crates.io/crates/frunk

Like these other crates, deflect *also* defines a reflection trait. However, deflect's trait is implemented for *all* types. Yes, all of them. The entire definition looks like this:
```rust
/// A reflectable type.
pub trait Reflect {
    /// Produces an ID that uniquely identifies the type in its compilation unit.
    #[inline(never)]
    fn local_type_id(&self) -> usize {
        <Self as Reflect>::local_type_id as usize
    }
}

// Implement `Reflect` for ALL types.
impl<T: ?Sized> Reflect for T {}
```
So how does deflect know anything about the internal structure of Rust types?

In a sense, it pretends it's a native debugger like [GDB] or [LLDB] --- it leverages [DWARF] debug info, emitted by rustc, to interpret the structure of Rust data.

[GDB]: https://www.sourceware.org/gdb/
[LLDB]:https://lldb.llvm.org/
[DWARF]: https://en.wikipedia.org/wiki/DWARF

When you call `.reflect` on a `dyn Reflect` value, deflect figures out its concrete type in four steps:
1. invokes `local_type_id` to get the memory address of your value's static implementation of `local_type_id`
2. maps that memory address to an offset in your application's binary
3. searches your application's debuginfo for the entry describing the function at that offset
4. parses that debugging information entry (DIE) to determine the type of `local_type_id`'s `&self` parameter.

The DIE of the `Self` type has information about the kind of the type (struct, enum, pointer, primitive, etc.), its size, alignment, and the locations and types of its fields. With all this, deflect is able to dynamically reflect the structure of your value's bytes.

In a future blog post, I'll write about how deflect also determines the concrete types of `dyn Trait` objects where the trait *doesn't* have a `local_type_id` method. I think the approach could be useful to native debuggers, like GDB and LLDB.

## Why!?
Last October, I [released async-backtrace](https://tokio.rs/blog/2022-10-announcing-async-backtrace), a crate for partially reflecting the logical structure of idle asynchronous tasks. Like [bevy_reflect] and [frunk], async-backtrace relies on proc-macro magic to produce its backtraces. Its traces *only* include functions annotated with `#[async_backtrace::framed]`; e.g.:
```rust
// included in traces
#[async_backtrace::framed]
async fn foo() {
    bar().await;
}

// NOT included in traces
async fn bar () {}
```

Predictably, few people relish the prospect of annotating *every* single async function in their codebase --- to say nothing of annotating async functions in the crates they depend on!

It was around this time that I learned that [Tyler Mandry][tmandry] was [experimenting with DWARF-directed async backtraces][zulip], capable of providing traces of arbitrary `async fn`s! His proof of concept was compelling, but generalizing it to *arbitrary* `Future`s (particularly, manually-implemented ones) would require the ability to reflect *arbitrary* types. Thus, deflect was born.

[tmandry]: https://github.com/tmandry
[zulip]: https://rust-lang.zulipchat.com/#narrow/stream/187312-wg-async/topic/Async.20stack.20trace.20support.20library/near/306350471

## Caveats Abound, Contributors Needed
Deflect, today, is a compelling, albeit unpolished, proof-of-concept. It is capable of reflecting over nearly all compositions of Rust types, but caveats abound. To name a few:
- It only works if DWARF debug info is packed into the application's binary.
    - It does not *yet* work on macOS, which splits debug info into other files.
    - It may never work on Windows, which does not use DWARF to encode debug info. 
- It does not *yet* specially handle the presence of `UnsafeCell`, which can lead to unsoundness.
- I am dissatisfied with how Rust's many reference types are currently reflected. Deflect must accurately mimic the nuances of field projection in Rust.
- How rustc encodes debug info is subject to change over time, and this crate only reflects rustc's conventions on a best-effort basis.
    - Relatedly, this crate needs to be tested against a far greater variety of Rust types!

I need help with all these challenges, and more!

## Finally, Thanks
Kudos to Tyler Mandry and Michael Woerister for their time, ideas and expertise. And, thanks to my employer, the Rust Platform Team at AWS, for supporting this work. I'm endlessly grateful to have the freedom and opportunity to push the bounds of what's possible in Rust.
