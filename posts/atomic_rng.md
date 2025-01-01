---
author: ville
date: 2024-12-31 12:21:00 +0200
categories: rngs rust
title: A Rust RNG with atomic state
---

{% include enable_mathjax.html %}

I've been working on a simple PRNG that requires only an immutable reference to use:

```rust
// Create a new RNG
let rng = Rng::new();

// A function that takes a reference to the RNG
//
//   look mom, not &mut 👇!
fn find_answer(thoughts: &Rng) {
    match thoughts.random() {
        42 => println!("Got 42! The answer!"),
        x => println!("Got {x}, not the answer"),
    }
}
```

Neat, huh? No more passing around mutable references to the RNG like a peasant! You can even make it a static and share it across threads:

```rust
static RNG: LazyLock<Rng> = LazyLock::new(Rng::new);
// ...
fn think() {
    thread::scope(|s| {
        (0..4).for_each(|_| {
            s.spawn(|| find_answer(&RNG));
        });
    });
}
think();
```

This is possible by making the RNG's state atomic.

## Implementation

The RNG iterates its state over the [Weyl sequence](https://en.wikipedia.org/wiki/Weyl_sequence) $x_i = x_{i-1} + c \mod 2^{64}$, returning the previous state hashed with [wyhash](https://github.com/wangyi-fudan/wyhash). The state is stored as an [`AtomicU64`](https://doc.rust-lang.org/std/sync/atomic/struct.AtomicU64.html) value:

```rust
pub struct Rng {
    /// The current state of the RNG.
    state: AtomicU64,
}
```

The RNG produces `u64`s by updating the state with a single `fetch_add` operation, followed by a call to `wyhash`:

```rust
impl Rng {
    /// The increment for the Weyl sequence.
    const INCREMENT: u64 = 0x9E3779B97F4A7FFF;

    /// Returns the next `u64` value from the pseudorandom sequence.
    fn u64(&self) -> u64 {
        // Read the current state and increment it
        let old_state = self.state.fetch_add(Self::INCREMENT, Ordering::Relaxed);

        // Hash the old state to produce the next value
        wyhash(old_state)
    }
    //...
}
```

The `fetch_add` call loads the current state, increments it, and stores the new value in a single atomic operation. The `Ordering::Relaxed` means that we don't care about the ordering of the operations, only that they are atomic. And that's it.

## Quality and performance

I think the RNG is okay to use if you don't need cryptographic security and you don't mind the performance penalty of using atomics.

Wyhash isn't cryptographically secure, but it passes [PractRand](http://pracrand.sourceforge.net/) pre0.95 at least up to 256 GB of generated data (I didn't have the patience to run it longer). The speed is okay, but not great. On my machine, the throughput of the atomic RNG is about 3.7 GB/s, compared to 7.7 GB/s for a variant that uses a mutable reference and no atomics. You can check out the [repository](https://github.com/koskinev/randy) on GitHub if you want to run the tests yourself.

## Code

A minimal implementation is just over 60 lines of code with comments. You can try it on the [Rust playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&code=use+std%3A%3Async%3A%3A%7Batomic%3A%3A%7BAtomicU64%2C+Ordering%7D%2C+LazyLock%7D%3B%0Ause+std%3A%3Ahash%3A%3A%7BBuildHasher%2C+RandomState%7D%3B%0Ause+std%3A%3Athread%3B%0A%0Apub+struct+Rng+%7B%0A++++%2F%2F%2F+The+current+state+of+the+RNG.%0A++++state%3A+AtomicU64%2C%0A%7D%0A%0Astatic+RNG%3A+LazyLock%3CRng%3E+%3D+LazyLock%3A%3Anew%28Rng%3A%3Anew%29%3B%0A%0A%23%5Binline%5D%0Afn+wyhash%28value%3A+u64%29+-%3E+u64+%7B%0A++++%2F%2F+These+constants%2C+like+the+%60INCREMENT%60+constant%2C+are+coprime+to+2%5E64.%0A++++const+ALPHA%3A+u128+%3D+0x11F9ADBB8F8DA6FFF%3B%0A++++const+BETA%3A+u128+%3D+0x1E3DF208C6781EFFF%3B%0A%0A++++let+mut+tmp+%3D+%28value+as+u128%29.wrapping_mul%28ALPHA%29%3B%0A++++tmp+%5E%3D+tmp+%3E%3E+64%3B%0A++++tmp+%3D+tmp.wrapping_mul%28BETA%29%3B%0A++++%28%28tmp+%3E%3E+64%29+%5E+tmp%29+as+_%0A%7D%0A%0Aimpl+Rng+%7B%0A++++%2F%2F%2F+The+increment+for+the+Weyl+sequence.%0A++++const+INCREMENT%3A+u64+%3D+0x9E3779B97F4A7FFF%3B%0A%0A++++%2F%2F%2F+Initializes+a+new+RNG.%0A++++pub+fn+new%28%29+-%3E+Self+%7B%0A++++++++let+seed+%3D+RandomState%3A%3Anew%28%29.hash_one%28%22foo%22%29%3B%0A++++++++let+state+%3D+AtomicU64%3A%3Anew%28seed%29%3B%0A++++++++Self+%7B+state+%7D%0A++++%7D%0A%0A++++%2F%2F%2F+Returns+the+next+%60u64%60+value+from+the+pseudorandom+sequence.%0A++++pub+fn+random%28%26self%29+-%3E+u64+%7B%0A++++++++%2F%2F+Read+the+current+state+and+increment+it%0A++++++++let+old_state+%3D+self.state.fetch_add%28Self%3A%3AINCREMENT%2C+Ordering%3A%3ARelaxed%29%3B%0A%0A++++++++%2F%2F+Hash+the+old+state+to+produce+the+next+value%0A++++++++wyhash%28old_state%29%0A++++%7D%0A%0A%7D%0A%0Afn+main%28%29+%7B%0A++++%2F%2F+++look+mom%2C+not+%26mut+%F0%9F%91%87%21%0A++++fn+find_answer%28thoughts%3A+%26Rng%29+%7B%0A++++++++match+thoughts.random%28%29+%7B%0A++++++++++++42+%3D%3E+println%21%28%22Got+42%21+The+answer%21%22%29%2C%0A++++++++++++x+%3D%3E+println%21%28%22Got+%7Bx%7D%2C+not+the+answer%22%29%2C%0A++++++++%7D%0A++++%7D%0A++++%0A++++fn+think%28%29+%7B%0A++++++++thread%3A%3Ascope%28%7Cs%7C+%7B%0A++++++++++++%280..4%29.for_each%28%7C_%7C+%7B%0A++++++++++++++++s.spawn%28%7C%7C+find_answer%28%26RNG%29%29%3B%0A++++++++++++%7D%29%3B%0A++++++++%7D%29%3B%0A++++%7D%0A++++think%28%29%3B%0A%7D):

```rust
use std::sync::{atomic::{AtomicU64, Ordering}, LazyLock};
use std::hash::{BuildHasher, RandomState};
use std::thread;

pub struct Rng {
    /// The current state of the RNG.
    state: AtomicU64,
}

static RNG: LazyLock<Rng> = LazyLock::new(Rng::new);

#[inline]
fn wyhash(value: u64) -> u64 {
    // These constants, like the `INCREMENT` constant, are coprime to 2^64.
    const ALPHA: u128 = 0x11F9ADBB8F8DA6FFF;
    const BETA: u128 = 0x1E3DF208C6781EFFF;

    let mut tmp = (value as u128).wrapping_mul(ALPHA);
    tmp ^= tmp >> 64;
    tmp = tmp.wrapping_mul(BETA);
    ((tmp >> 64) ^ tmp) as _
}

impl Rng {
    /// The increment for the Weyl sequence.
    const INCREMENT: u64 = 0x9E3779B97F4A7FFF;

    /// Initializes a new RNG.
    pub fn new() -> Self {
        let seed = RandomState::new().hash_one("foo");
        let state = AtomicU64::new(seed);
        Self { state }
    }

    /// Returns the next `u64` value from the pseudorandom sequence.
    pub fn random(&self) -> u64 {
        // Read the current state and increment it
        let old_state = self.state.fetch_add(Self::INCREMENT, Ordering::Relaxed);

        // Hash the old state to produce the next value
        wyhash(old_state)
    }

}

fn main() {
    //   look mom, not &mut 👇!
    fn find_answer(thoughts: &Rng) {
        match thoughts.random() {
            42 => println!("Got 42! The answer!"),
            x => println!("Got {x}, not the answer"),
        }
    }
    
    fn think() {
        thread::scope(|s| {
            (0..4).for_each(|_| {
                s.spawn(|| find_answer(&RNG));
            });
        });
    }
    think();
}
```

## Further reading

- [Atomics](https://doc.rust-lang.org/nomicon/atomics.html) in the Rustonomicon
- [How to Test with PractRand](https://www.pcg-random.org/posts/how-to-test-with-practrand.html) by O'Neill
- [testingRNG : testing popular random-number generators](https://github.com/lemire/testingRNG), a repository by Lemire
- [A Low Discrepancy Shuffle Iterator (+Random Access & Inversion)](https://blog.demofox.org/2024/05/19/a-low-discrepancy-shuffle-iterator-random-access-inversion/) by demofox
