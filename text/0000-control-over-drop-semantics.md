- Feature name: Control over Drop Semantics
- Start date: 2026-08-27
- Project Goal: https://rust-lang.github.io/rust-project-goals/2026/manually-drop-attr.html
- WIP POC: [[POC] create an MVP for using `Destruct` for custom dtors](https://github.com/rust-lang/rust/pull/156090)

## Summary

Currently, it is impossible to alter or replace the recursive calling of field destructors on user-defined types in Rust.
The current workaround is to wrap a type's fields in `ManuallyDrop` to prevent the inner types' destructors from being called recursively,
and then manually drop them using `ManuallyDrop::drop` (equivalent to `ptr::drop_in_place`) in a `Drop` implementation for the type.

This proposal would allow instead implementing the `Destruct` trait with its `drop_in_place` method to customize the behavior when an object is dropped.
When the user does not provide an implementation for `Destruct::drop_in_place`, the regular drop behavior for the type is preserved:
a call to any `Drop::drop`, followed by the recursive destruction of each of its fields.

## Motivation: Status Quo with `ManuallyDrop` - C++ Compatibility Hazards
Using `ManuallyDrop` on fields makes object construction more verbose,
but more importantly it raises compatibility hazards for bindings that expose foreign (e.g. C++) types in Rust.
For fields that should be destroyed by C++ code,
Rust code must wrap each with `ManuallyDrop` to prevent the containing type from destroying the child field recursively.
In C++, adding a destructor to a type is not generally considered a breaking change,
but doing so would introduce bugs in mixed-language settings if Rust bindings do not simultaneously add `ManuallyDrop` wrappers around any fields having that type.

To make this situation less fraught, it is beneficial to be able to fully replace the Rust destruction behavior.
In the case of C++ bindings, the Rust destruction behavior would simply call into the C++ destructor.

## Guide-level explanation

The `core::mem::Destruct` now exposes the following interface, which will be the entry point for all automatic object destruction:
```rust
trait Destruct {
    unsafe fn drop_in_place(&mut self);
}
```

This method has a default implementation that runs `Drop::drop` before dropping each field, which was the behavior prior to this RFC.

If an explicit implementation of the trait provides a different implementation, it will replace the default destruction behavior for the type.

The following diagram shows how `Destruct`'s default implementation can be understood for an example type:
```
                              ┌─────────────────────────────────────────────────────────────┐
                              │ Destruct::drop_in_place(self: &mut Self) {                  │
struct BoxedFd {              │     Drop::drop(self); // A custom impl could not call this! │
    boxed_fd: Box<i32>,       │     core::intrinsics::drop_fields_in_place(self);           │
}                             │ }                                                           │
                              └──────────────┬────────────────┬─────────────────────────────┘
                                 calls first │                │
                              ┌──────────────┘                │  calls second
                              │                               └─────────────────┐
             ┌────────────────V───────────────┐                                 │
             │ impl Drop for BoxedFd {        │       ┌─────────────────────────V────────────────────────────┐
             │     fn drop(&mut self) {       │       │ "drop glue"                                          │
             │         close(*self.boxed_fd); │       │ core::intrinsics::drop_fields_in_place(&mut self) {  │
             │     }                          │       │     Destruct::drop_in_place(&raw self.boxed_fd)      │
             │ }                              │       │ }                                                    │
             └────────────────────────────────┘       └──────────────────────────────────────────────────────┘
```

Note that:
  1. The default behavior runs the `Drop` impl for the type before dropping individual fields.
     An explicit implementation of `Destruct` could not call into `Drop`, because explicitly invoking `.drop()` is forbidden.
     As such, it is an error to implement both `Destruct` and `Drop` for the same type.
  2. `core::intrinsics::drop_fields_in_place` intrinsic does not actually exist.
     It might be convenient to add this to expose only the recursive portion of drop glue, but this is not strictly
     necessary; a user can also manually call `Destruct::drop_in_place` on each field in sequence.

### An Example
To demonstrate how the new API would be used, the following example:

```rust
#![feature(const_destruct)]
use std::marker::Destruct;

struct A {
    a: String,
}

impl Destruct for A {
    unsafe fn drop_in_place(_to_drop: &mut Self) {
        println!("Hey i was dropped");
    }
}

fn main() {
    let a = A { a: String::from("some string data") };
}
```

would have the output

```bash
Hey i was dropped
```

and leak the memory of the `String`.

Calling `Destruct::drop_in_place(&mut _to_drop.a)` in `A`'s `drop_in_place` method would result in the `String` being properly cleaned up.

## Compiler Changes and Implementation Strategy

During trait resolution, if the compiler sees that there are no user implementations of `Destruct` when it searches for all candidates implementations of the trait, only then does it consider the builtin implementation (the traditional drop glue that drops each field) as a candidate. Otherwise, the user implementation is used and the builtin implementation is not generated.

## Details to decide

1. Whether the compiler should directly insert the drop glue into the trait method, or have it do so in a `core::intrinsic` shim and have a blanket implementation for `Destruct` that calls this intrinsic. So the new `drop_in_place` function could look like

```rust
#[lang = "destruct_drop_in_place"]
unsafe fn drop_in_place(to_drop: *mut Self) {
    core::intrinsics::drop_in_place_shim(to_drop);
}
```

## FAQs

1. What's the difference between `Drop` and `Destruct`?

- `Drop` is closer to a normal trait in that it may be implemented to specify user-defined cleanup for a type, which is strictly additive to the built-in per-field cleanup performed by "drop glue". The magic part is that when a value goes out of scope, the first step performed by the `core::ptr::drop_in_place` intrinsic is to call `Drop::drop` for your type if `Drop` is implemented, before the rest of the drop behavior kicks in (in particular, "drop glue" that cleans up the type's fields).
- `Destruct` is today a fully builtin trait, much like `Sized`. It is implemented for basically every type in existence. It serves one purpose today: while every type implements `Destruct`, not all of them implement `const Destruct`. They do if all the `Drop` impls involved in dropping the given type (i.e., recursively so) are `const`. So this trait serves to characterize the constness of the drop behavior for a type, but does not currently have a way to alter that behavior.

2. Why is this implemented on `Destruct` instead of `Drop`?

This function is implemented on the `Destruct` trait as it is exactly the trait whose bounds capture whether "this type can go out of scope" and thus what dictates what happens when the type goes out of scope, which is what this RFC suggests to make user-controllable. This is further reinforced `Destruct` being the trait bound for `core::ptr::drop_in_place`. In the future, if these two traits are merged, this distinction will be simplified away.

3. Why involve a second trait at all? Couldn't this be folded into `Drop`?

In theory, it could, but `Destruct` is already around as a way to discuss the drop behavior of types,
and the `Drop` trait has some historic infelicities, such as being usable as a trait bound but only in ways that are surprising.
For example, `Vec<T>` implements `Drop` but `String` does not! The `Drop` bound does not mean "can this type be dropped", nor does it mean "does this type perform cleanup when dropped". At best, it means "is there specific user-specified impl of the `Drop` trait for this type", which is tautological and not a question about semantic *behavior*.

The `Destruct` trait (`std::marker::Destruct`), on the other hand, is a specialized marker trait for items that can be dropped. It currently exists to tell the compiler whether the recursive act of dropping (including running generated "drop glue") is permissible within const evaluation.

Further cleanups of these traits would be desirable, but are outside the scope of this RFC.

## Real-life examples

### Changing Drop Order

Rust currently guarantees that fields are dropped in their declaration order.
Users can reorder fields to change this ordering (independently from changing the type's in-memory layout, if `#[repr(C)]` is not applied),
but readers of Rust code are not used to the order fields being significant, and this hangs significant semantics off of a usually-ignored aspect of syntax.
It is also possible to drop order by wrapping fields in `ManuallyDrop`, but this does not solve the problem in an open-shut fashion:
the user must then write a custom `Drop` impl that drops those fields, which is not necessarily easy to do correctly.
In particular, this impl is unlikely to exactly replicate the built-in automatic destruction behavior with respect to avoiding leak amplification.
(The default destruction behavior temporarily catches the first panic that may occur when dropping fields, and before re-raising it, continues to drop fields, immediately aborting if a second field's drop implementation panics.)
As such, it is no longer recommended (see [i.rlo discussion](https://internals.rust-lang.org/t/need-for-controlling-drop-order-of-fields/12914)
and the diff on [rust-lang/rust PR #76150](https://github.com/rust-lang/rust/pull/76150))
to use ManuallyDrop for this purpose (though the Rustonomicon contains an
[outdated such suggestion](https://doc.rust-lang.org/nomicon/dropck.html#a-related-side-note-about-drop-order)).
One reason for this is that if a custom `Drop` impl does not switch from unwinding to aborting for the second panic, some fields will simply be leaked,
which can be a soundness issue if a type which is `Pin` has its destructor skipped
(see [Pin's documentation](https://doc.rust-lang.org/std/pin/index.html#subtle-details-and-the-drop-guarantee:)).

So we take it as granted that some types will have side-effects in their `Drop` implementations that might be mediated through I/O or FFI,
and their relative destruction order will not be a concern of the borrow checker
(if they interacted solely through Rust references, we would be in [dropck territory](https://doc.rust-lang.org/nomicon/dropck.html)).
Thus an example might look like this, where external concerns require "Foo dropped" to be printed before "Bar dropped":

```rust
use std::marker::Destruct;

struct Foo;
impl Drop for Foo {
    fn drop(&mut self) {
        println!("Foo dropped")
    }
}
struct Bar;
impl Drop for Bar {
    fn drop(&mut self) {
        println!("Bar dropped")
    }
}

// Normally, fields would be dropped in declaration order, with `bar` dropped before `foo`.
struct HoldsBoth {
    bar: Bar,
    foo: Foo,
}

// Drop `foo` before `bar`.
impl Destruct for HoldsBoth {
    unsafe fn drop_in_place(to_drop: &mut Self) {
        Destruct::drop_in_place(&mut to_drop.foo);
        Destruct::drop_in_place(&mut to_drop.bar);
    }
}
```
Prints "Foo dropped" and then "Bar dropped".

### Calling C++ Destructors

It is UB to destroy C++ objects by simply destroying each of their fields the way Rust would,
if the C++ destructor has side effects that the program depends on.
See the [C++20 Standard, \[basic.life\] p5](https://timsong-cpp.github.io/cppwp/n4868/basic.life#5).
C++ classes that do not replace the default destructor (and as such have no side effects to their destruction)
may be soundly possible to destroy from Rust by destroying each field,
but destructors with side effects do exist and are an important use case for interoperability.

```rust
extern "C" { fn cpp_dtor_uring_state(this: *mut UringState); }

struct UringState {
    ring: Uring,
    buffers: [UringBuf; 16],
}

impl Destruct for UringState {
    unsafe fn drop_in_place(to_drop: &mut Self) {
        cpp_dtor_uring_state(to_drop);
    }
}
```

### Avoiding Stack Overflow Dropping Recursive ADTs

One use case for customizing destruction behavior for types would be to avoid an undesirable attributes of the default behavior,
its propensity for stack overflow when dropping deeply nested recursive data structures.

This is explored in the paper ["Efficient Deconstruction with Typed Pointer Reversal (abstract)" by Munch-Maccagnoni and Donence](https://hal.science/hal-02177326).

While they note (§3.1) that their technique is not always applicable in Rust without changing the definition of types
(because there may not be enough bits available in enum tags to track the necessary intermediate states of cleanup),
this limitation could be overcome with explicit opt-in from the user, or possibly with an attribute macro on the type definition.

#### Destructuring/Pattern Matching

But they mention another limitation which suggests that it would be useful for this proposal to *not* forbid
pattern matching on types with custom destruction behavior:

> Rust also disables pattern-matching on owned values with custom destructor,
when our algorithm is meant as a drop-in replacement of the default one.
For all these reasons, some form of compiler support seems highly desirable.

Indeed, allowing to specify a drop-in replacement of the default destruction behavior of Rust types is exactly what this proposal is about,
and the `Drop` trait already allows types to opt out of destructuring pattern matching,
so it seems advantageous to ensure that implementing the `Destruct` trait does not prevent types from being destruct*ured*.

Furthermore, [previous discussion of C++ interop for destruction](https://github.com/rust-lang/lang-team/issues/135)
touched on the desirability of still being able to pattern-match on types even if their destruction behavior matches C++.

## Appendix: A leak amplification example

A panicking destructor does not prevent other destructors in the same scope from running (see [rust-lang/rust#14875](https://github.com/rust-lang/rust/issues/14875)):
```rust
struct HoldsFooBar {
    foo: Foo,
    bar: Bar,
}

struct Foo;

impl Drop for Foo {
    fn drop(&mut self) { panic!(); }
}

struct Bar;
impl Drop for Bar {
    fn drop(&mut self) { println!("Bar dropped"); }
}

fn main() {
    let _hfb = HoldsFooBar { foo: Foo, bar: Bar };
}
```
This example prints "Bar dropped" after the panic from `Foo::drop` is displayed.

## Drawbacks

This does add more conceptual branches into answering "what happens when a value goes out of scope?", but possibly in a way that is more teachable:
we can point to the `Destruct` trait's default behavior as where drop glue actually happens, which was more implicit previously.

Terminologically, this is kind of a mess. I would really like to have a more formal delineation
between `Drop`, `Destruct`, destruct*uring* "destruction", "destructors", and "drop glue".

## Alternatives

- Stick with the current design requiring `ManuallyDrop` on fields.
  This has the ergonomic and maintenance costs noted for C++ code, and requiring `Drop` prevents destructuring,
  which is a blocker for the "efficient drops"/avoiding stack overflow use case for ADTs.
- Implement with an attribute to opt out of destruction of individual fields or all fields,
  rather than enhancing the `Destruct` trait.
  Custom cleanup logic would then go in the `Drop` trait, which is undesirable for reasons discussed in the FAQ.
  Or maybe we could introduce an entirely separate trait for such cleanup,
  but that seems like a proliferation of magic traits since `Drop` and `Destruct` both already exist.
