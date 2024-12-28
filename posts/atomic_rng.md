---
author: ville
date: 2024-12-27 11:17:00 +0200
categories: rngs rust
title: A RNG with atomic state
---

{% include enable_mathjax.html %}

We'll implement a standalone `random` function in Rust you can use like this:

```rust
use randy::random;

fn find_answer() -> Option<u64> {
    match random() {
        42 => Some(42),
        _ => None,
    }
}
```

Unlike [`rand::random`](https://docs.rs/rand/latest/rand/fn.random.html), our version doesn't initialize a new PRNG every time you call it. Instead, it uses a single PRNG with *static* state. The idea is to store the state of the PRNG as an atomic value, so you can access it from anywhere and update it without needing mutable access.

## The wyrand PRNG

I'll use [`wyrand`](https://github.com/wangyi-fudan/wyhash), because it needs only 64 bits of state, making it trivial to update atomically. The algorithm is not cryptographically secure, but apparantely it [passed](https://github.com/lemire/testingRNG?tab=readme-ov-file#visual-summary) BigCrush and PractRand, so it shouldn't be terrible. A minimal implementation is below:

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

It took me a while to figure how the magic number ensures that the state will iterate through all possible values before repeating, but a post about low discrepancy sequences on [demofox.org](https://blog.demofox.org/2024/05/19/a-low-discrepancy-shuffle-iterator-random-access-inversion/) and the Wikipedia article on [coprime integers](https://en.wikipedia.org/wiki/Coprime_integers) made it click. The constant $$a=\text{MAGIC_ADD}$$ is coprime with $$2^{64}$$, which means that the least common multiple of $$a$$ and $$2^{64}$$ is the product of the two, i.e. $$a \cdot 2^{64}$$. This means that $$a \cdot x \mod 2^{64}$$ will will be nonzero, unless $$x$$ is a multiple of $$2^{64}$$.

## Going atomic

I'll skip the stuff about [atomics](https://doc.rust-lang.org/nomicon/atomics.html) to keep this short. (Also, I probably should understand the topic a little better to be able to explain it.) The only thing we need to know is that since an atomic value is guaranteed to be updated in a single step, you don't need to have a mutable reference to the value to update it.

```rust
use std::sync::atomic::{AtomicU64, Ordering};

let x = AtomicU64::new(1);
x.fetch_add(1, Ordering::Relaxed);
print!("{}", x.load(Ordering::Relaxed));
```

So let's make the state a static.

```rust
static STATE: AtomicU64 = {
    let seed = {
        let hasher = std::hash::RandomState::new();
        hasher.hash_one("foo")
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

To work around this, we need to initialize the state lazily. We can do this by using a [`LazyLock`](https://doc.rust-lang.org/std/sync/struct.LazyLock.html), stabilized in Rust 1.80:

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

Setting `#[cfg(debug_assertions)]` makes the state deterministic in debug builds, which is useful for testing. Now, we can implement the `random()` function:

```rust
pub fn random() -> u64 {
    // fetch_add increments the state and returns the old value.
    let old_state = STATE.fetch_add(MAGIC_ADD, std::sync::atomic::Ordering::Relaxed);
    wymix(old_state.wrapping_add(MAGIC_ADD))
}
```

This only returns `u64`s, but can be adapted to return other types. That's it!  Below is the full example code, which you can try out in the [playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&code=mod+randy+%7B%0A++++use+std%3A%3Async%3A%3A%7Batomic%3A%3A%7BAtomicU64%2C+Ordering%7D%2C+LazyLock%7D%3B%0A++++use+std%3A%3Ahash%3A%3ABuildHasher%3B%0A++++%0A++++const+MAGIC_ADD%3A+u64+%3D+0x_2d35_8dcc_aa6c_78a5%3B%0A++++const+MAGIC_XOR%3A+u64+%3D+0x_8bb8_4b93_962e_acc9%3B%0A++++%0A++++static+STATE%3A+LazyLock%3CAtomicU64%3E+%3D+LazyLock%3A%3Anew%28%7C%7C+%7B%0A++++++++let+seed+%3D+%7B%0A++++++++++++let+hasher+%3D+std%3A%3Ahash%3A%3ARandomState%3A%3Anew%28%29%3B%0A++++++++++++hasher.hash_one%28%22foo%22%29%0A++++++++%7D%3B%0A++++++++AtomicU64%3A%3Anew%28seed%29%0A++++%7D%29%3B%0A++++%0A++++fn+wymix%28x%3A+u64%2C+y%3A+u64%29+-%3E+u64+%7B%0A++++++++let+%28mut+a%2C+mut+b%29+%3D+%28x%2C+x+%5E+y%29%3B%0A++++++++let+r+%3D+%28a+as+u128%29+*+%28b+as+u128%29%3B%0A++++++++%28a%2C+b%29+%3D+%28r+as+u64%2C+%28r+%3E%3E+64%29+as+u64%29%3B%0A++++++++a+%5E+b%0A++++%7D%0A++++%0A++++%2F%2F+Returns+a+pseudorandom+%60u64%60%0A++++pub+fn+random%28%29+-%3E+u64+%7B%0A++++++++%2F%2F+Increment+the+state+and+read+the+old+value%0A++++++++let+old_state+%3D+STATE.fetch_add%28MAGIC_ADD%2C+Ordering%3A%3ARelaxed%29%3B%0A++++++++wymix%28old_state.wrapping_add%28MAGIC_ADD%29%2C+MAGIC_XOR%29%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++use+randy%3A%3Arandom%3B%0A%0A++++for+_+in+0..10+%7B%0A++++++++println%21%28%22%7B%7D%22%2C+random%28%29%29%3B%0A++++%7D%0A%7D):

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

    for _ in 0..10 {
        println!("{}", random());
    }
}
```
