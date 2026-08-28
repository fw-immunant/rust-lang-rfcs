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

Currently, to customize the automatic portion of drop behavior for a type `T`, we have to wrap its fields in `ManuallyDrop` to prevent their destructors from being called,
and then manually drop them using `ManuallyDrop::drop` (equivalent to `ptr::drop_in_place`) in a `Drop` implementation for the type.

This proposal would allow instead implementing the `Destruct` trait with its `drop_in_place` method to customize the behavior when an object is dropped.
When the user does not provide an implementation for `Destruct::drop_in_place`, the regular drop behavior for the type is preserved:
a call to any `Drop::drop`, followed by the "drop glue" that drops each of its fields.
For example,

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

has the output

```bash
Hey i was dropped
```

## Changes and how it works

If the compiler sees that there are no user implementations of Destruct while its finding the candidates for the implementatoin, only then does it assemble builtin candidates for `Destruct`. Otherwise, the user implementation is used and the builtin candidates are never assembled.

## Details to decide

1. Whether the compiler should straight-away insert the drop glue into the trait function, or have it do that in a `core::intrinsic` shim and have a blanket implementation for `Destruct` that calls this function. So the new `drop_in_place` function looks like

```rust
#[lang = "destruct_drop_in_place"]
unsafe fn drop_in_place(to_drop: *mut Self) {
    core::intrinsics::drop_in_place_shim(to_drop);
}
```

## FAQs

1. What's the difference between `Drop` and `Destruct`?

   The difference between Destruct and Drop is as follows:

- Drop is an almost normal trait: you can implement it for your type, and use it as a trait bound (tho that's discouraged); the magic part is that the core::ptr::drop_in_place intrinsic, which is called when a value goes out of scope, will call Drop::drop for your type if Drop is implemented
- Destruct is today a fully builtin trait, much like Sized. It is implemented for basically every type in existence. It serves one purpose today: while every type implements Destruct, not all of them implement const Destruct. They do if all the Drop impls involved in dropping the given type (i.e., recursively so) are const.

2. Why is this impemented on `Destruct` instead of `Drop`?

This function is implemented on the `Destruct` trait as it is exactly the trait that says "this type can go out of scope", i.e. exactly the trait that allows you to call `drop_in_place`. In the future, if these two get merged, its even better (fingers crossed).

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
