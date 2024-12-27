---
layout: post
title: "Atomic integers and RNGs"
date: 2024-12-27 11:17:00 -0000
categories: rngs rust
---

# Atomic integers and random numbers

In this post, I'll show how to implement a global PRNG you can use by just calling `random()`, like this:

```rust
use randy::random;

fn find_answer() -> Option<u64> {
    match random() {
        42 => Some(42),
        _ => None,
    }
}
```

The PRNG will not be cryptographically secure, but it will be relatively fast and good enough to be useful. The idea is to store the state of the PRNG as a static atomic value, so you can access it from anywhere and update it without needing mutable access.

## The `wyrand` PRNG

I'll use [`wyrand`](https://github.com/wangyi-fudan/wyhash), because it needs only 64 bits of state, making it trivial to update atomically. The algorithm is not cryptographically secure, but it passed BigCrush and PractRand, so it shouldn't be terrible. A minimal implementation of wyrand is:

```rust
const MAGIC_ADD: u64 = 0x_2d35_8dcc_aa6c_78a5;
const MAGIC_XOR: u64 = 0x_8bb8_4b93_962e_acc9;

fn wymix(x: u64, y: u64) -> u64 {
    let (mut a, mut b) = x, x ^ y;
    let r = (a as u128) * (b as u128);
    a, b = (r as u64, (r >> 64) as u64);
    a ^ b
}

/// A pseudorandom number generator that uses the wyrand algorithm.
pub struct WyRng {
    /// The current state.
    state: u64,
}

impl WyRng {
    /// Creates a new `WyRng` with the given seed.
    pub fn new(seed: u64) -> Self {
        Self { state: wymix(seed, MAGIC_XOR) }
    }

    /// Returns the next pseudorandom number from the generator.
    pub fn u64(&mut self) -> u64 {
        // The magic number is coprime with 2^64, so the state will 
        // iterate through all 2^64 possible values before repeating.
        self.state = self.state.wrapping_add(MAGIC_ADD);
        wyhash(self.state)
    }
}
```

## Going atomic

To keep it short, I'll skip the stuff about [atomics](https://doc.rust-lang.org/nomicon/atomics.html). The only thing we need to know is that an atomic value can be updated even if it's not mutable:

```rust
use std::sync::atomic::{AtomicU64, Ordering};

let x = AtomicU64::new(1);
x.fetch_add(1, Ordering::Relaxed);
print!("{}", x.load(Ordering::Relaxed));
```

So let's make the state a static!  

```rust
static STATE: AtomicU64 = {
    let seed = {
        let hasher = std::hash::RandomState::new();
        hasher.hash_one('foo')
    };
    AtomicU64::new(seed)
};
```

Unfortunately, this fails to compile:

```text
error[E0015]: cannot call non-const fn `RandomState::new` in statics
  --> src/main.rs:10:26
   |
10 |             let hasher = std::hash::RandomState::new();
   |                          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

To work around this, we need to initialize the state lazily. We can do this by using a `LazyLock`, stabilized in Rust 1.80:

```rust
static STATE: LazyLock<AtomicU64> = LazyLock::new(|| {
        #[cfg(not(debug_assertions))]
        let seed = {
            let hasher = std::hash::RandomState::new();
            hasher.hash_one("foo")
        };
        #[cfg(debug_assertions)]
        AtomicU64::new(1234)
    });
```

The `#[cfg(debug_assertions)]` makes the state deterministic in debug builds, which is useful for testing. Now, we can implement the `random()` function:

```rust
pub fn random() -> u64 {
    // fetch_add increments the state and returns the old value.
    let old_state = STATE.fetch_add(MAGIC_ADD, std::sync::atomic::Ordering::Relaxed);
    wymix(old_state.wrapping_add(MAGIC_ADD))
}
```

This only returns `u64`s, but can be adapted to return other types. That's it!  Below is the full example code, which you can try out in the [playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=91d1e4e8ef32aea94993ea78be600d0a):

```rust
mod randy {
    use std::sync::{atomic::{AtomicU64, Ordering}, LazyLock};
    use std::hash::BuildHasher;
    
    const MAGIC_ADD: u64 = 0x_2d35_8dcc_aa6c_78a5;
    const MAGIC_XOR: u64 = 0x_8bb8_4b93_962e_acc9;
    
    static STATE: LazyLock<AtomicU64> = LazyLock::new(|| {
        let seed = {
            let hasher = std::hash::RandomState::new();
            hasher.hash_one("foo")
        };
        AtomicU64::new(seed)
    });
    
    fn wymix(x: u64, y: u64) -> u64 {
        let (mut a, mut b) = (x, x ^ y);
        let r = (a as u128) * (b as u128);
        (a, b) = (r as u64, (r >> 64) as u64);
        a ^ b
    }
    
    // Returns a pseudorandom `u64`
    pub fn random() -> u64 {
        // Increment the state and read the old value
        let old_state = STATE.fetch_add(MAGIC_ADD, Ordering::Relaxed);
        wymix(old_state.wrapping_add(MAGIC_ADD), MAGIC_XOR)
    }
}

fn main() {
    use randy::random;

    let vals = [random(), random()];
    println!("{vals:?}");
}
```
