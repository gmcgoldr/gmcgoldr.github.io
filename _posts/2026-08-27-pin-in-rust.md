---
layout: default
title:
author: null
---

# Pin in Rust: What, Why, and How

When working with async Rust, you'll sometimes encounter types like `Pin<Box<dyn Future<...>>>`, and you might be tempted to look the other way. These inscrutable types instill a small sense of dread in me: what kind of footguns might be lurking in this part of the language that I don't understand? I set out to explain this for myself, and the outcome of this exploration is this blog post. The aim of this post is to introduce the `Pin` type so that when you next encounter such a type you don't have to look the other way.

Say we have a type `DontMoveThis` and for some reason (which we will motivate shortly when we talk about `Future`s), we don't want values of this type to move. That is to say, we want them to stay at the same memory address. How can we do this?

```rust
let value: DontMoveThis = DontMoveThis {};
let moved_value = value;   // uh-oh
```

As a first attempt we could try putting it in a `Box`. `Box` has a useful property: the value it holds stays at the same heap address even when the `Box` itself is moved.

```rust
let value = DontMoveThis {};
let boxed: Box<DontMoveThis> = Box::new(value);
let moved_box = boxed;  // `value` address didn't change
```

You'll notice that we *did* move the `value` into the `Box`. This is an important finding: if values of a some type can *never ever* move, then those values would be border-line useless: you couldn't even return them from a function. Typically what you want is a value that can't move after a certain event in its lifecycle. E.g. a value that doesn't move after initialization, or after being moved into a `Box`.

We've now found a way to build a value, and after some setup it can be moved around in its `Box` without changing its memory address. Problem solved, right?

```rust
let moved_value = *boxed;  // uh-oh
```

The problem is that while the value won't move while it's inside the `Box`, there's nothing to prevent the box's owner from moving the value out of the box. If `DontMoveThis` has to *really* not move (i.e. this is an invariant that can't be violated), `Box` isn't sufficient. We need some way to *pin* the value in place so that it can't be removed from the box. Let's try something else.

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
let moved_value = *pinned.boxed;  // won't compile, `boxed` is private
let borrowed: &DontMoveThis = &pinned.boxed;  // also won't compile :(
```

That works: there's no way to access the boxed value from the `pinned` instance, so there's no way to move the value out! But it's also not very useful, the value is effectively gone, we might as well have called this `BlackHole`. Still, maybe we're on the right path. Let's try something a bit different this time.

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
let moved_value = *pinned.boxed;  // won't compile, `boxed` is private
let not_moved_ref = pinned.get();  // yay!
```

This time, we've found a way to pin a value that can still be accessed through a reference. Moving `pinned` will move `pinned.boxed`, but that won't move the boxed value. Our `PinBox` gives access to neither an owned `T` nor an `&mut T`, it's not possible to move the value out of `pinned`. It's worth thinking about this a little bit: there's no way to move a value out of its memory location in safe Rust without owning the value or having a mutable reference to it. If an `&T` could move the value it refers to, Rust’s distinction between shared and exclusive access would fall apart.

At this point you have a rudimentary understanding of the `Pin` type: conceptually it works very much like `PinBox` but with more bells and whistles. If you want to stop reading now, you've gotten 80% of the understanding by reading about 20% of the post, not bad! But there are a few wrinkles that we should address before calling it a day. As with all things, they of course come with added complexity (hint: if you want to understand the `Unpin` marker trait you should keep reading).

The first wrinkle is that the author of `DontMoveThis` might need to write code that mutate the pinned value. But once the value is pinned, there's no way to get a mutable reference, by design.

```rust
struct DontMoveThis {
    value: i32,
}

impl DontMoveThis {
    fn set_value(pinned: &mut PinBox<Self>, value: i32) {
        pinned.get().value = value;  // won't compile
    }
}
```

On way around this is to use interior mutability, which allows mutations of fields inside `DontMoveThis` without requiring an `&mut DontMoveThis`.

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

But forcing authors of pinned types to implement every mutation using interior mutability is too constraining. Some operations just need `&mut T`. We can provide an escape hatch: the author of `DontMoveThis` might know how to mutate the type without moving it. This would be unsafe territory, so let's give unsafe access to `&mut T`.

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

We now have a `PinBox` struct that allows a caller to pin some value after an initialization step. The value continues to be readable with safe code, and can be mutated with interior mutability or through the unsafe escape hatch.

Here's the second wrinkle: say we have a trait implemented by `DontMoveThis` but also by `ThisCanMove`. If the trait has a method that mutates needs to mutate `Self`, it will need to accept `&mut PinBox<Self>` instead of `&mut Self` so that it can be implemented for pinned instances of `DontMoveThis`:

```rust
trait Settable: Sized {
    fn set_value(pinned: &mut PinBox<Self>, value: i32);
}
```

We just saw how to implement `set_value` for `DontMoveThis`. The author of `DontMoveThis` expects to use the unsafe escape hatch because they are working on a type whose pinning invariant must be preserved. But now we're forcing the author of `ThisCanMove` to also use an unsafe escape hatch to do something that is completely safe for a type that has no address-sensitive invariants.

This is the rather narrow reason for the `Unpin` marker trait. `Unpin` means that a type does not care about the pinning guarantee, so `PinBox` can safely allow ordinary mutable access to it.

Most Rust types are `Unpin` automatically. A type that relies on pinning must instead opt out of `Unpin`. We can represent that in our example with `PhantomPinned`:

```rust
use std::marker::PhantomPinned;

struct DontMoveThis {
    value: i32,
    _pin: PhantomPinned,
}
```

We can now let `Unpin` types bypass the `PinBox` restrictions:

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

Here's a quick look at how `Pin` works with `Unpin` just to help this sink in.

| | **`T: Unpin`** | **`T: !Unpin`** |
| -- | -- | -- |
| **`Pin<Box<T>>`** | `T` can move   | `T` can't move |
| **Bare `T`**      | `T` can move   | `T` can move |

This might seem a bit perplexing. What's the point of having `Pin` if a type can just opt out of its restrictions? As the caller of `Pin`, maybe you want to make *really sure* the value you put in there doesn't move regardless of its type. In that case you're not the target audience of `Pin`. Note that our `PinBox` implementation is valid, so you could use that without implementing `get_mut` to enforce this restriction for arbitrary types.

It's time to make good on my promise from the start of this post and finally get to `Future`s. This will also answer two unresolved questions: who is `Pin` for, and why would someone need to make a type immovable?

Without doing a deep dive on futures, what's important to understand is that each async function returns its own compiler-generated `Future` type that stores the function's state across await points.

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

And herein lies the entire motivation for `Pin`: a `Future` that holds the state of an async function will sometimes need to contain references into itself. Let's take a quick detour and try to write our own self-referential type to see where this goes wrong.

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

Somewhat surprisingly, this works (I read at least one blog post that claims this doesn't work). It might seem that the borrow checker should prevent self-references because they _can_ give rise to dangling pointers: if `value` moves, then `value.data` moves and `value.reference` would point to the old address. But _can_ and _will_ are two different things and the borrow checker makes that distinction. It knows that as long as `value` doesn't move, there is no dangling pointer. So it doesn't need to prevent us from making the self-reference, it just needs to prevents us from moving `value`.

At first glance this seems like a good thing: the borrow checker just did the work to make `value` immovable. But not in the way we want. `value` can't be moved at all while that borrow exists: not even into a `Box`.

If you're thinking a few steps ahead, you might realize that `value` isn't address-sensitive when `reference: None`, and became immovable only *after* we set the self-reference. What about trying to move it into a box *before* setting the reference?

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

That won't work either because while *we* know that moving the `Box` won't move the `SelfReferential` value it points to, the borrow checker doesn't know this. As far as it knows, moving the box _might_ move the value it points to. So it does the safe thing and prevents us from moving the entire box.

How do we get around this restriction? We use unsafe code to hide the self-reference from the borrow checker, usually by representing it with a raw pointer rather than a Rust reference. And since we lose the borrow checker's guarantee that the reference can't dangle, we have to replace it with another invariant: once that self-reference exists, the value must remain pinned.

Back to ~~the~~ `Future`s: the interesting part of `Future`s is the `poll` trait method, and understanding it will also shed light on how a type author enforces pinning. As the author of `DontMoveThis`, you woule need to make your type `!Unpin`, and you would also need APIs that enforce pinning once the value becomes address-sensitive.

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

The important difference is that `poll` doesn't receive `&mut Self`; it receives `Pin<&mut Self>`.

Now try to construct a situation in safe Rust where you move the address of a `!Unpin` `Future` after polling has put it into an address-sensitive state. You can't. Calling `poll` requires the future to already be held by `Pin`. Once polling establishes a self-reference, the `Pin` contract requires that the future remain at that address from then on.

Before I can finally explain `Pin<Box<dyn Future<...>>>`, there are two small details left. `Pin` is generic over pointer-like types such as `Box<T>` and `&mut T`. The important indirection is that `Pin<Ptr>` doesn't pin the pointer `Ptr`; it pins the pointee that `Ptr` dereferences to. So `Pin<Box<T>>` pins the `T`. As for the `dyn` part, that's because each async function has its own concrete compiler-generated `Future` type. So a type that can represent any such `Future` will be over `dyn Future` or `impl Future`.

That pretty much wraps it up! If you made it this far, there's a satisfying conclusion that I hope you can draw from this: pinning in Rust doesn't rely on compiler magic. Pinning falls out of the normal type system and is implemented as a standard library feature. If you're interested in the language developer perspective, I recommend this first-hand take on Pin's development [here](https://without.boats/blog/pin/).

In the end, the solution that `Pin` offers manages to concentrate the complexity of address-sensitive types into a few types and functions. Some of the alternatives to `Pin` would have "spread" that complexity throughout the language. No single aspect of those alternatives would've necessarily seemed more complex, but as a developer your working mental model of the compiler and type system would have to be a bit bigger even when touching parts of Rust that have little to do with `Future`s and address-sensitive data.
