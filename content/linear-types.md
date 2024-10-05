+++
title = "Designing with Linear Types"
date = 2024-10-04
+++

Linear types are types that *must* be moved from or consumed in some way exactly once. These would fit into Rust's type system pretty nicely, but not perfectly. Lots of smart people have written lots of words about what the API might look like, potential footguns and designs to mitigate them, and plenty of other considerations. Rather than try to add to that, I'm going to write some fantasy Rust that uses the feature and see how it looks!

<!-- more -->

We can call it a motivating example if that makes it sound more productive than fantasizing. At any rate, in order to fantasize, we have to guess at what the API will look like. A simple-looking one is just to add a `struct` that can't be dropped. This means you could stick one into your own `struct` and *it* couldn't be dropped either, or at least, not in the default way: that would force you to handle it explicitly. The API might look something like this:

```rs
pub struct NoDrop;

impl NoDrop {
    pub fn drop_explicit(self) { /* compiler-internal stuff */ }
}
```

Now let's say we have a network connection with all `async` methods, and we want the elusive async drop behavior. If you're unfamiliar, sometimes closing a connection involves asynchronous operations like sending network messages or awaiting the responses. If you want to do this in the destructor to ensure it always happens, you have to fall back on blocking APIs since you can't `.await`. This is not ideal. You can close the connection outside of the destructor, but then a) the destructor has nothing to do, and b) the client can easily forget to call `close`. Linear types force us to use the type, and forbid us from implicitly dropping it, so we can do something like this:

```rs
pub struct SomeNetworkConnection {
    internal_data: InternalData,
    _no_drop: NoDrop,
}

impl SomeNetworkConnection {
    pub async fn close(self) {
        let Self { internal_data, _no_drop } = self;
        finalize_connection(&internal_data).await; // idk what this does
        _no_drop.drop_explicit(); // If we leave this off, we get a compiler error!
    }
}
```

And just like that, we need to close this thing before it goes out of scope:

```rs
async fn bad() {
    let mut connection = SomeNetworkConnection::open().await;
    connection.send(PAYLOAD).await;
} // comipler error: `connection` can't be dropped due to `NoDrop` member

async fn good() {
    let mut connection = SomeNetworkConnection::open().await;
    connection.send(PAYLOAD).await;
    // We consume `connection` here, meaning it doesn't get dropped at the end of the scope!
    connection.close().await;
}
```

This API works, but leaves some holes: what happens if `connection.send` panics? Normally `connection` would be dropped as the stack is unwound, but what if it *can't* be dropped? What does the standard library look like? With `NoDrop`, suddenly you've got all sorts of containers that can't be dropped naively when they're generic over a linear type. These problems have been explored pretty exhaustively elsewhere, but for the purposes of fantasizing, let's come up with a kind-of solution:

```rs
pub struct SomeNetworkConnection {
    internal_data: InternalData,
}

impl !Drop for SomeNetworkConnection {
    fn panic_drop(self) {
        let Self { .. } = self;
    }
}
```

This borrows heavily from Niko Matsakis' [article on must-move types](https://smallcultfollowing.com/babysteps/blog/2023/03/16/must-move-types). The difference I've added here is a small bit of fantasizing that I don't think the Rust compiler would accept at the moment: `panic_drop`. We've essentially turned `!Drop` into a contract that says "this type can't be implicitly dropped, but the compiler still needs to know how to safely get rid of it when unwinding". Depending on the type, the implementation could be a no-op, an abort, a core dump, etc.

Since we've gotten rid of the `NoDrop` member, you might think we could now implement `close` like this:

```rs
impl SomeNetworkConnection {
    pub async fn close(self) {
        finalize_connection(&self.internal_data).await;
    }
}
```

But that would leave us with `self` rolling off the edge of the function, triggering a drop! If you're like me and haven't thought about this too hard, it might look like a chicken-egg problem: we can't drop implicitly, so we need to define some explicit drop routine, but any function we write needs to deal with that `self` parameter without implicitly dropping it. The solution is to do the same thing we did with the `NoDrop` implementation: destructure `self`:

```rs
impl SomeNetworkConnection {
    pub async fn close(self) {
        let Self { internal_data } = self;
        finalize_connection(&internal_data).await;
    }
}
```

Now there's no `SomeNetworkConnection` to drop because we've poofed it out of existance. We've turned it into a disassembled pile of its constituent parts, each of which *can* be implicitly dropped. Niko Matsakis hit on this in the aforementioned article, saying that Rust's visibility system plays very nicely with linear types: a linear type must be destructured in order to get rid of it, but a `struct` with private fields must be destructured by code in the same `mod`! `SomeNetworkConnection` is such a `struct`, and so it must provide some sort of destructor -- even an `async` one! On the other hand, you can make a `struct` `!Drop` simply by adding a `!Drop` member to it. If such a `struct` had only public fields, it would be the client's responsibility to eventually destructure it, then deal with each `!Drop` field individually. I find a lot of elegance in how consistent this would make the language:

```rs
// The only way to make a struct is to construct it like this:
let x = SomeStruct { a, b, c };
// The only way to get rid of a linear struct is to destructure it like this:
let SomeStruct { a, b, c } = x;
```

So far we've used linear types to invent `async` destructors. What else we can do? One of my favorite games is FTL: Faster Than Light. You command a spaceship with multiple systems like engines, weapons and shields. You have a limited number of discrete power intervals to allocate to each system. Power allocated to a system is available only to that system until you reallocate it, removing it from the system it's in and putting it in another. One implementation might look like this:

```rs
pub struct Power {
    reactor: u8,
    system_power: HashMap<System, u8>,
}

enum System {
    Engines,
    Weapons,
    Shields,
}

impl Power {
    pub fn add_power(&mut self, system: System) {
        if let Some(remaining_reactor) = self.reactor.checked_sub(1) {
            self.system_power.get_mut(&system).unwrap() += 1;
            self.reactor = remaining_reactor;
        }
    }

    pub fn remove_power(&mut self, system: System) {
        if let Some(remaining_system) = self.system_power[&system].checked_sub(1) {
            self.reactor += 1;
            *self.system_power.get_mut(&system).unwrap() = remaining_system;
        }
    }
}
```

Here I'm using `checked_sub` because I cleverly realized that if I don't bounds-check this, a user could crash the game by attempting to remove power that doesn't exist -- or worse, when it's compiled in release, it'll bug out and give them infinite power! Well, this is a simple example, but there's always room for bugs. They creep in, especially when you refactor, and predicting bugs is hard, otherwise I'd be out of a job. On the other hand, Rust's type system can all but eliminate certain classes of bugs by introducing static restrictions, moving checks from runtime to compile time. Is there a way to turn some of this into compile-time checks?

As it turns out, linear types provide an alternate design. What if we model discrete power bars as instances of a linear type?

```rs
pub struct PowerBar;

impl !Drop for PowerBar {}

pub struct Power {
    reactor: Vec<PowerBar>,
    system_power: HashMap<System, Vec<PowerBar>>,
}
```

Instead of storing power levels as integers, we now store them as a collection of discrete objects. We create a fixed number of power bars at initialization and put them in `reactor`. When the user wants to reallocate power, we shuffle them around:

```rs
impl Power {
    pub fn add_power(&mut self, system: System) {
        if let Some(power_bar) = self.reactor.pop() {
            self.system_power.get_mut(&system).unwrap().push(power_bar);
        }
    }

    pub fn remove_power(&mut self, system: System) {
        if let Some(power_bar) = self.system_power.get_mut(&system).unwrap().pop() {
            self.reactor.push(power_bar);
        }
    }
}
```

Now we literally can't add power to a system unless there's reactor power available. We also don't need to worry about underflowing integers: we'd have to do some whacky stuff with `unsafe` to get this to misbehave. Now, you'll notice linear types don't actually make this possible: the trick is just that `PowerBar` can't be `Clone` or `Copy`. However, with all that power shuffling, it's possible that you could accidentally drop a `PowerBar` -- meaning your ship would have less total power to work with. Linear types prevent this: if you accidentally forget to `push` a `pop`ped `PowerBar` onto a new power stack, the compiler will kindly inform you of this fact.

As an aside, at first I cringed at the idea of using a `Vec` where a `u8` would suffice. After some thinking, I realized that a `Vec` generic over a zero-sized type will not actually trigger any allocations. This means it will essentially optimize down to a single `usize` being incremented and decremented as power is moved around -- exactly what the initial implementation looked like. In theory, the compiler could realize that the other fields are useless and optimize them out, but I don't think that'd happen in practice.

At the end of the day, this isn't particularly ground-breaking, and arguably would scare people to look at: "why are we storing a `Vec` in here? Shouldn't that just be an integer?" What interests me is the ability of the compiler to enforce user-defined invariants. This program would make it very easy to reason about how many power bars exist. You can gate creation of `PowerBar` behind a specific API on `Power` and make sure it's only being called when, say, the player upgrades their reactor. You could potentially *lose* those power bars -- they get stuck in a place you didn't realize they could. Making `PowerBar` private to the `power` module would make it so that `PowerBar` *can't even leave the module!* Other scopes can't fiddle with it because it doesn't exist as far as they're concerned! Put this all together and I'd expect this system to be pretty bulletproof (ignoring my blatant `unwrap` calls in the above examples -- how hypocritical of me). Power is only created when the user upgrades their reactor. Power can only be in the reactor or a system. We can't accidentally increment a counter without decrementing another. Power is only destroyed when the ship is destroyed. If you want to remove a system from the ship, the compiler will complain at you: "you can't drop this system, it has a `Vec<PowerBar>` in it!" Oh, right, better put those back in the reactor. And just like that, the compiler has prevented a bug: losing overall power when removing a still-powered system.

At the other end, I put a lot of design work into what is ultimately a pretty trivial system.
