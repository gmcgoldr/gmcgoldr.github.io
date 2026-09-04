---
layout: default
title: Why Does Rust Need Send and Sync?
author: null
---

# Why Does Rust Need Send and Sync?

In this post, I want to shed some light on two [marker traits](https://doc.rust-lang.org/std/marker/index.html) in Rust: the [`Send`](https://doc.rust-lang.org/std/marker/trait.Send.html) and [`Sync`](https://doc.rust-lang.org/std/marker/trait.Sync.html) traits.

When working with multithreaded Rust, you'll often see `Send` and `Sync`. You can usually gloss over them: most of the time, it's enough to know they have something to do with using data across threads. But that can leave an uncomfortable gap: here are parts of the type system you rely on without really understanding what guarantees they're giving you. This post will try to close that gap.

First, a quick word on marker traits: these are traits that encode properties for the type system. In the case of `Send` and `Sync`, those properties allow the compiler to enforce invariants about how values can be used across threads. Unlike most traits, the interesting thing about a marker trait isn't some behaviour it provides (there is none), but what its presence tells the compiler about a type.

Part of what makes `Send` and `Sync` hard to reason about is that they live at the boundary between *unsafe* and *safe* Rust, a boundary you usually don't think about when working in safe Rust. Unsafe code can step outside some of the rules safe Rust enforces, such as by allowing mutation through shared references. To do that safely, the implementation has to rely on other invariants instead. When those invariants concern moving or sharing values across threads, `Send` and `Sync` are how Rust enforces them.

The best way to understand `Send` and `Sync` is to look at a type that is neither `Send` nor `Sync`. `Send` and `Sync` are auto traits: most types get `Send` and `Sync` automatically from the types they contain. The interesting cases are types implemented using unsafe Rust, where `!Send` or `!Sync` is needed to preserve the implementation's safety invariants. `Rc` is the canonical example in the standard library: it is both `!Send` and `!Sync`. (`impl !Send` and `impl !Sync` are currently nightly-only features for user code.) Looking at why gives us a concrete way to understand what each trait means.

Let's start with `!Send`. Consider the following snippet:

```rust
use std::rc::Rc;
use std::thread;

let x1 = Rc::new(42);
let x2 = x1.clone();

thread::scope(|s| {
    // With `move`, the closure takes ownership of `x2`.
    s.spawn(move || {
        let _ = x2.clone();
    });

    let _ = x1.clone();
});
```

This code doesn't compile, because `Rc` is `!Send`: moving `x2` into the spawned thread isn't allowed.

`Rc`'s reference counting isn't atomic. `x1` and `x2` share the same reference count. When `x2` is cloned in the spawned thread, `x1` can be cloned concurrently in the main thread. If those increments overlap, both threads try to update the same non-atomic reference count. That would be a data race, and the reference count could end up incorrect.

This is exactly the kind of safe/unsafe boundary we talked about above. Multiple `Rc` clones share the same reference count, and each can mutate it. Safe Rust's borrowing rules wouldn't allow that mutation through shared references, so this ultimately relies on unsafe Rust. But, unlike `Arc`, `Rc` uses non-atomic operations on its reference count. Accessing it concurrently from different threads would be unsafe. `Rc` therefore needs the type system to enforce one restriction: **an `Rc` value must never be moved to another thread**. This is what `!Send` prevents.

But moving an `Rc` isn't the only way another thread could get access to the reference count. Consider what happens when we share a reference to the `Rc` instead:

```rust
use std::rc::Rc;
use std::thread;

let x1 = Rc::new(42);

thread::scope(|s| {
    // Without `move`, the closure borrows `x1`.
    s.spawn(|| {
        let _ = x1.clone();
    });

    let _ = x1.clone();
});
```

This also doesn't compile, because `Rc` is `!Sync`.

Here we never move an `Rc` to another thread. Instead, the spawned thread accesses `x1` through a shared reference. When it clones `x1`, the main thread can clone `x1` concurrently. If those increments overlap, both threads again try to update the same non-atomic reference count.

`Rc` therefore needs the type system to enforce another restriction: **shared references to an `Rc` must never be used from multiple threads**. This is what `!Sync` prevents.

`!Send` prevents an `Rc` itself from being moved to another thread. `!Sync` prevents shared references to an `Rc` from being shared between threads. Together, they preserve the invariant that `Rc`'s non-atomic reference count is never accessed concurrently from multiple threads.
