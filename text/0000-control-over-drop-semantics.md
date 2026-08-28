---
# System prepended metadata

title: Control over Drop semantics Pre-rfc

---

# Control over Drop semantics Pre-rfc

> [!info] Metadata
> Project Goal: https://rust-lang.github.io/rust-project-goals/2026/manually-drop-attr.html
>
> (very rough) POC: [[POC] create an MVP for using `Destruct` for custom dtors](https://github.com/rust-lang/rust/pull/156090)

## Core Idea

Currently, it is impossible to alter or replace the recursive calling of field destructors on user-defined types in Rust.
The current workaround is to wrap a type's fields in `ManuallyDrop` to prevent the inner types' destructors from being called recursively,
and then manually drop them using `ManuallyDrop::drop` (equivalent to `ptr::drop_in_place`) in a `Drop` implementation for the type.

This proposal would allow instead implementing the `Destruct` trait with its `drop_in_place` method to customize the behavior when an object is dropped.
When the user does not provide an implementation for `Destruct::drop_in_place`, the regular drop behavior for the type is preserved:
a call to any `Drop::drop`, followed by the recursive destruction of each of its fields.

### Status Quo with `ManuallyDrop`: C++ Compatibility Hazards
Using `ManuallyDrop` on fields makes object construction more verbose,
but more importantly it raises compatibility hazards for bindings that expose foreign (e.g. C++) types in Rust.
For fields that should be destroyed by C++ code,
Rust code must wrap each with `ManuallyDrop` to prevent the containing type from destroying the child field recursively.
In C++, adding a destructor to a type is not generally considered a breaking change,
but doing so would introduce bugs in mixed-language settings if Rust bindings do not simultaneously add `ManuallyDrop` wrappers around any fields having that type.

To make this situation less fraught, it is beneficial to be able to fully replace the Rust destruction behavior.
In the case of C++ bindings, the Rust destruction behavior would simply call into the C++ destructor.

### An Example
To demonstrate how the new API would be used, the following example:

```rust
#![feature(const_destruct)]
use std::marker::Destruct;
struct A {
    a: String,
}

impl Destruct for A {
    unsafe fn drop_in_place(_to_drop: *mut Self) {
        println!("Hey i was dropped");
    }
}

fn main() {
    let a = A { a: String::new() };
}
```

would have the output

```bash
Hey i was dropped
```

and leak the memory of the `String`.

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

### Changing drop order

```rust
use std::marker::Destruct;

struct Context;

struct Resource<'a> {
    context: &'a Context,
}

struct Data<'a> {
    context: Context,
    resource: Resource<'a>,
}

impl Destruct for Data {
    unsafe fn drop_in_place(to_drop: *mut Self) {
        Destruct::drop_in_place(to_drop.resource);
        Destruct::drop_in_place(to_drop.context);
    }
}
```

### Calling C++ Destructors

```rust
struct UringState {
    ring: Uring,
    buffers: [UringBuf; 16],
}

impl Destruct for UringState {
    unsafe fn drop_in_place(to_drop: *mut Self) {
        cpp_dtor_uring_state(to_drop);
    }
}
```

## Common Questions and Answers

### What even is this `Destruct` trait?

The `Destruct` trait (`std::marker::Destruct`) is a specialized marker trait for items that can be dropped. It exists to tell the compiler whether the recursive act of dropping (including generated "drop glue") is permissible within const evaluation.

### How does this differ from `Drop`?

The `Drop::drop` method is currently called from `ptr::drop_in_place` when a value goes out of scope:wq
