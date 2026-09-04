---
layout: default
title: Pinning Down Rust’s Pin
author: null
---

# Pinning Down Rust’s Pin

When working in async Rust, I sometimes encounter `Pin<Box<dyn Future<...>>>` types, and until now I haven’t really understood what they mean: what does `Pin` say about the `Future`? Why `Box`? What can I do with this type, and more importantly, what can’t I do with it?

This post is my attempt to answer those questions. We’ll build a simplified version of `Pin` from scratch to understand how it works and what it enforces.

Say we have a type `DontMoveThis` and, for some reason, we need its values to stay at the same memory address.

> These kinds of values are called *address-sensitive* and usually come about because they hold a pointer to one of their own fields.

```rust
let value = DontMoveThis {};
let moved_value = value; // uh-oh
```

We moved `value` into `moved_value`, so its address could have changed. A move doesn’t necessarily change a value’s address, but to guarantee that it remains fixed, we must prevent the value from moving. How do we do that?

As a first attempt, we could put the value in a `Box`. Creating the box moves the value into a heap allocation, but after that, moving the `Box` moves the owner without moving the value stored in that allocation.

```rust
let value = DontMoveThis {};
let boxed: Box<DontMoveThis> = Box::new(value);
let moved_box = boxed;  // box moved, but not value!
```

> You might notice that we _did_ move `value` into the `Box`. A value that can _never ever move_ isn’t very useful: it couldn’t even be returned from a function. A value typically becomes address-sensitive only after some step in its lifecycle, such as setting a pointer. But `DontMoveThisAfterSomeStep` is too long for a blog post.

Great, now we can move the `Box` without moving the value it contains. Problem solved, right?

```rust
let moved_value = *boxed;  // uh-oh
```

The problem is that while moving the `Box` won’t move the value inside it, there’s nothing to prevent the box’s owner from moving the value out of the box. If `DontMoveThis` has to *really* not move (i.e. this is an invariant that can’t be violated), `Box` isn’t sufficient. We need some way to *pin* the value in place so that it can’t be removed from the box. Let’s try something else.

```rust
mod pinbox {
    pub struct PinBox<T> {
        boxed: Box<T>,
    }

    impl<T> PinBox<T> {
        // swallow the value to pin
        pub fn new(value: T) -> Self {
            Self { boxed: Box::new(value) }
        }
    }
}

let value = DontMoveThis {};
let pinned = pinbox::PinBox::new(value);
let moved_value = *pinned.boxed;  // won’t compile, `boxed` is private
let borrowed: &DontMoveThis = &pinned.boxed;  // also won’t compile :(
```

It works! But only because we’ve lost all access to the value. We can’t move it, but we can’t do anything else with it either. We might as well have called it `BlackHole`. Still, maybe we’re on the right path. Let’s try something a bit different.

```rust
mod pinbox {
    pub struct PinBox<T> {
        boxed: Box<T>,
    }

    impl<T> PinBox<T> {
        pub fn new(value: T) -> Self {
            Self { boxed: Box::new(value) }
        }

        // give access to &T but not to the bare value
        pub fn get(&self) -> &T {
            &self.boxed
        }
    }
}

let value = DontMoveThis {};
let pinned = pinbox::PinBox::new(value);
let moved_value = *pinned.boxed;  // won’t compile, `boxed` is private
let not_moved_ref = pinned.get();  // yay!
```

This looks much better: the value still can’t be moved, and now we can read it. In safe Rust, moving a value out of its memory location requires an owned `T` or an `&mut T`, and `PinBox` exposes neither.

> It’s worth thinking about why this works: if an `&T` could move the value it refers to, Rust’s distinction between shared and exclusive access would fall apart.

At this point you have a rudimentary understanding of the `Pin` type: conceptually it works very much like `PinBox` but with more bells and whistles. If you want to stop reading now, you’ve gotten 80% of the understanding by reading about 20% of the post. Not bad! But there are a few wrinkles that we should address before calling it a day.

The first wrinkle is that the author of `DontMoveThis` might need to write code that mutates the pinned value. But once the value is pinned, there’s no way to get a mutable reference, by design.

```rust
struct DontMoveThis {
    value: i32,
}

impl DontMoveThis {
    fn set_value(pinned: &mut PinBox<Self>, value: i32) {
        pinned.get().value = value;  // won’t compile
    }
}
```

One way around this is to use interior mutability, which allows mutations of fields inside `DontMoveThis` without requiring an `&mut DontMoveThis`.

```rust
struct DontMoveThis {
    value: Cell<i32>,
}

impl DontMoveThis {
    fn set_value(pinned: &mut PinBox<Self>, value: i32) {
        pinned.get().value.set(value);  // yay!
    }
}
```

But forcing authors of pinned types to implement every mutation using interior mutability is too constraining. Some operations just need an `&mut T`. We can provide an escape hatch: unsafe access to `&mut T`. This is unsafe because Rust can no longer guarantee that the value won’t move; we have to rely on the author to use the `&mut T` without violating that invariant.

This might seem a bit ridiculous: after all this work, we’re reintroducing the very thing we were trying to prevent, access to `&mut T`. But alas, this is how `Pin` works: it preserves the invariant in safe Rust, while unsafe code gets both the flexibility and the responsibility to uphold it.

```rust
impl<T> PinBox<T> {
    pub unsafe fn get_unchecked_mut(&mut self) -> &mut T {
        &mut self.boxed
    }
}

struct DontMoveThis {
    value: i32,
}

impl DontMoveThis {
    fn set_value(pinned: &mut PinBox<Self>, value: i32) {
        let this: &mut Self = unsafe { pinned.get_unchecked_mut() };
        this.value = value;
    }
}
```

We now have a `PinBox` that lets the caller pin a value at some step in its lifecycle. After that, the value must stay put, but it can still be read with safe code and mutated using interior mutability or the unsafe escape hatch.

Here’s the second wrinkle: say we have a trait implemented by both `DontMoveThis` and `ThisCanMove`. A trait that wants to mutate `Self` will often have a method that takes `&mut Self`. But that method can’t be used once a value is pinned. If we want it to work with pinned values, its argument has to change:

```rust
trait Settable: Sized {
    // `&mut Self` becomes `&mut PinBox<Self>`
    fn set_value(pinned: &mut PinBox<Self>, value: i32);
}
```

We just saw how to implement this for `DontMoveThis`: mutating an address-sensitive value after it is pinned means using the unsafe escape hatch. That makes sense here. But now every implementer of `Settable` has to use the escape hatch. For `ThisCanMove`, which has no address-sensitive invariant, that `unsafe` is completely unnecessary.

This is the rather narrow reason for the `Unpin` marker trait. That might feel a little unsatisfying, but as we’ll see when we get to `Future`s, this use case comes up all the time in async Rust.

`Unpin` lets `ThisCanMove` bypass the unsafe escape hatch altogether and get an `&mut T` in safe Rust. This might seem even more ridiculous: now we’re allowing safe access to the very thing we were trying to prevent? But alas, `Pin` is only about preventing safe access to `&mut T` for types that declare themselves address-sensitive by being `!Unpin`.

> `!Unpin` is a bit of a double negative. `Unpin` is an auto trait, so most types implement it by default. Address-sensitive types are the exception: they opt into the pinning restriction by opting out of `Unpin`.

We can make `DontMoveThis` opt out by adding `PhantomPinned`:

```rust
use std::marker::PhantomPinned;

struct DontMoveThis {
    value: i32,
    // this is how to make the type `!Unpin`
    _pin: PhantomPinned,
}
```

We can encode that rule in `PinBox`:

```rust
mod pinbox {
    pub struct PinBox<T> {
        boxed: Box<T>,
    }

    impl<T> PinBox<T> {
        pub fn new(value: T) -> Self {
            Self { boxed: Box::new(value) }
        }

        pub fn get(&self) -> &T {
            &self.boxed
        }

        pub unsafe fn get_unchecked_mut(&mut self) -> &mut T {
            &mut self.boxed
        }
    }

    // this method exists only for types that are Unpin
    impl<T: Unpin> PinBox<T> {
        pub fn get_mut(&mut self) -> &mut T {
            &mut self.boxed
        }
    }
}

struct ThisCanMove {
    value: i32,
}

impl Settable for ThisCanMove {
    fn set_value(pinned: &mut PinBox<Self>, value: i32) {
        // this is like having a parameter `&mut Self` with one extra step
        let this: &mut Self = pinned.get_mut();
        this.value = value;
    }
}
```

Here’s a quick look at how `Pin` works with `Unpin` just to help this sink in.

| | **`T: Unpin`** | **`T: !Unpin`** |
| -- | -- | -- |
| **`Pin<Box<T>>`** | `T` can move   | `T` can’t move |
| **Bare `T`**      | `T` can move   | `T` can move |

This might seem a bit perplexing. What’s the point of having `Pin` if a type can just opt out of its restrictions? As the caller of `Pin`, maybe you want to make *really sure* the value you put in there doesn’t move, regardless of its type. But you’re not the target audience of `Pin`. `Pin` was designed for authors of address-sensitive types (i.e. `Future`s, as we’ll see shortly) who need safe Rust to preserve their invariants.

> You might also wonder: what’s the point of `Pin` if the caller can opt out by not pinning a `!Unpin` value? The caller doesn’t actually have that freedom. The type’s author will make the operation that puts the value into an address-sensitive state return a `Pin<Box<Self>>`. From then on, the caller can’t safely take the value out of the `Pin`; that’s where it lives out the rest of its lifecycle.

It’s time to make good on my promise from the start of this post and finally get to `Future`s. This will answer the remaining question: why would someone need to make a type immovable?

Without doing a deep dive on futures, what’s important to understand is that each async function returns its own compiler-generated `Future` type that stores the function’s state across await points.

```rust
async fn foo() {
    let data = 42;
    let reference = &data;

    do_something_async().await;

    println!("{reference}");
}
```

Conceptually, this function might return a hypothetical `Future` type like this:

```rust
enum FooFuture {
    Start,
    Waiting {
        data: i32,
        reference: *const i32, // points to `data`
    },
    Done,
}
```

And herein lies the entire motivation for `Pin`: a `Future` that holds the state of an async function will sometimes need to contain references into itself. Let’s take a quick detour and try to write our own self-referential type to see where this goes wrong.

```rust
struct SelfReferential<'a> {
    data: i32,
    reference: Option<&'a i32>,
}

let mut value = SelfReferential {
    data: 42,
    reference: None,
};

value.reference = Some(&value.data);
```

Somewhat surprisingly, this works (I read at least one blog post that claims this doesn’t work). It might seem that the borrow checker should prevent self-references because they _can_ give rise to dangling pointers: if `value` moves, then `value.data` moves and `value.reference` would point to the old address. But _can_ and _will_ are two different things, and the borrow checker makes that distinction. It knows that as long as `value` doesn’t move, there is no dangling pointer. So it doesn’t need to prevent us from making the self-reference. It just needs to prevent us from moving `value`.

Great, the borrow checker managed to make `value` immovable without any help from `Pin`! Wait, was all that work for nothing? Not quite: `value` can’t be moved anywhere while the borrow exists, not even into a `Box`. It’s not much use in this state: it’s effectively imprisoned in its local scope.

If you’re thinking a few steps ahead, you might notice that `value` becomes address-sensitive only after we set `reference`. What if we move it into a `Box` before that step in its lifecycle?

```rust
let value = SelfReferential {
    data: 42,
    reference: None,
};
let mut boxed = Box::new(value);
boxed.reference = Some(&boxed.data);

let moved_box = boxed;
// error: cannot move out of `boxed` because it is borrowed
```

That won’t work either because while *we* know that moving the `Box` won’t move the `SelfReferential` value it points to, the borrow checker doesn’t know this. As far as it knows, moving the box _might_ move the value it points to. So it does the safe thing and prevents us from moving the entire box.

How do we get around this restriction? We use unsafe code to hide the self-reference from the borrow checker, usually by representing it with a raw pointer rather than a Rust reference. And since we lose the borrow checker’s guarantee that the reference can’t dangle, we have to replace it with another invariant: once that self-reference exists, the value must remain pinned.

Back to ~~the~~ `Future`s: the interesting part is the `poll` trait method, and understanding it will also shed light on how a type author enforces pinning. As the author of `DontMoveThis`, you would need to make your type `!Unpin`, and you would also need APIs that enforce pinning once the value becomes address-sensitive.

Imagine for a moment that `Future::poll` accepted an ordinary `&mut self` instead of a `Pin<&mut Self>`:

```rust
pub trait Future {
    fn poll(&mut self);
}

let mut future = FooFuture::Start;
future.poll();
let mut moved_future = future; // move it somewhere else
moved_future.poll(); // uh-oh: `reference` is dangling
```

Here is the actual `Future` definition. Note that it looks a lot like our example `Settable` trait, with a few more `Future`-specific details:

```rust
pub trait Future {
    type Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

The important difference is that `poll` doesn’t receive `&mut Self`; it receives `Pin<&mut Self>`.

Now try to construct a situation in safe Rust where you move a `!Unpin` `Future` to a different address after polling has put it into an address-sensitive state. You can’t. Calling `poll` requires the future to already be held by `Pin`. Once polling establishes a self-reference, the `Pin` contract requires that the future remain at that address from then on.

Before I can finally explain `Pin<Box<dyn Future<...>>>`, there are two small details left. `Pin` is generic over pointer-like types such as `Box<T>` and `&mut T`. The important indirection is that `Pin<Ptr>` doesn’t pin the pointer `Ptr`; it pins the pointee that `Ptr` dereferences to. So `Pin<Box<T>>` pins the `T`. As for the `dyn` part, that’s because each async function has its own concrete compiler-generated `Future` type. So we need to refer to such a `Future` through `dyn Future` or `impl Future`.

That pretty much wraps it up! If you made it this far, there’s a satisfying conclusion here: pinning in Rust doesn’t rely on compiler magic. Pinning falls out of the normal type system and is implemented as a standard library feature. If you’re interested in a language developer’s perspective, I recommend this first-hand take on Pin’s development [here](https://without.boats/blog/pin/).

In the end, the solution `Pin` offers manages to concentrate the complexity of address-sensitive types into a few types and functions. Some of the alternatives to `Pin` would have "spread" that complexity throughout the language. No single aspect of those alternatives would’ve necessarily seemed more complex, but as a developer, your working mental model of the compiler and type system would have to be a bit bigger even when touching parts of Rust that have little to do with `Future`s and address-sensitive data.
