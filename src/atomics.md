# 원자들 (Atomics)

러스트는 원자들의 메모리 모델에 관해서는 꽤나 대놓고 C++20의 것을 그대로 이어받습니다. 그것은 이 모델이 특별히 대단하거나 이해하기 쉬워서가 아닙니다. 
오히려 이 모델은 꽤 복잡하고 [몇 가지 문제점이 있는 것으로][C11-busted] 알려져 있습니다. 그러나 이것은 *모두가* 원자들을 설계하는 것은 잘 못한다는 사실에 실리적으로 인정하는 것입니다. 
정말 최소한, 우리는 C/C++의 메모리 모델 주변에 있는 도구들과 연구들의 혜택을 받을 수 있습니다. (여러분은 이 모델을 "C/C++11" 혹은 그냥 "C11"로 부르는 것을 볼 것입니다. 
C는 그냥 C++의 메모리 모델을 복사합니다. 또한 C++11은 그 모델의 처음 버전이었지만 그 뒤로 몇 가지 버그 수정을 받았습니다.)

이 팩에서 그 모델을 전부 설명하는 것은 별로 희망이 없습니다. 실용적으로 제대로 이해하려면 하나의 책이 통째로 필요할 만큼, 사람을 정신 나가게 만드는 인과관계 그래프들로 정의되어 있거든요. 
만약 모든 자질구레한 세부사항을 알고 싶다면, [C++ 명세를][C++-model] 보셔야 할 겁니다. 그래도, 우리는 기본적인 것과 러스트 개발자들이 마주하는 몇 가지 문제들을 설명하려 노력할 것입니다.

C++ 메모리 모델은 근본적으로 우리가 원하는 의미와, 컴파일러가 원하는 최적화와, 우리의 하드웨어가 원하는 비일관적인 혼돈 사이에 다리를 놓고자 하는 것입니다. 
*우리는* 그저 프로그램을 짜고 정확히 우리가 시킨 대로 하게 할 것입니다, 다만 빠르게요. 좋을 것 같지 않습니까?

## 컴파일러 재배치

Compilers fundamentally want to be able to do all sorts of complicated
transformations to reduce data dependencies and eliminate dead code. In
particular, they may radically change the actual order of events, or make events
never occur! If we write something like:

<!-- ignore: simplified code -->
```rust,ignore
x = 1;
y = 3;
x = 2;
```

The compiler may conclude that it would be best if your program did:

<!-- ignore: simplified code -->
```rust,ignore
x = 2;
y = 3;
```

This has inverted the order of events and completely eliminated one event.
From a single-threaded perspective this is completely unobservable: after all
the statements have executed we are in exactly the same state. But if our
program is multi-threaded, we may have been relying on `x` to actually be
assigned to 1 before `y` was assigned. We would like the compiler to be
able to make these kinds of optimizations, because they can seriously improve
performance. On the other hand, we'd also like to be able to depend on our
program *doing the thing we said*.

## Hardware Reordering

On the other hand, even if the compiler totally understood what we wanted and
respected our wishes, our hardware might instead get us in trouble. Trouble
comes from CPUs in the form of memory hierarchies. There is indeed a global
shared memory space somewhere in your hardware, but from the perspective of each
CPU core it is *so very far away* and *so very slow*. Each CPU would rather work
with its local cache of the data and only go through all the anguish of
talking to shared memory only when it doesn't actually have that memory in
cache.

After all, that's the whole point of the cache, right? If every read from the
cache had to run back to shared memory to double check that it hadn't changed,
what would the point be? The end result is that the hardware doesn't guarantee
that events that occur in some order on *one* thread, occur in the same
order on *another* thread. To guarantee this, we must issue special instructions
to the CPU telling it to be a bit less smart.

For instance, say we convince the compiler to emit this logic:

```text
initial state: x = 0, y = 1

THREAD 1        THREAD 2
y = 3;          if x == 1 {
x = 1;              y *= 2;
                }
```

Ideally this program has 2 possible final states:

* `y = 3`: (thread 2 did the check before thread 1 completed)
* `y = 6`: (thread 2 did the check after thread 1 completed)

However there's a third potential state that the hardware enables:

* `y = 2`: (thread 2 saw `x = 1`, but not `y = 3`, and then overwrote `y = 3`)

It's worth noting that different kinds of CPU provide different guarantees. It
is common to separate hardware into two categories: strongly-ordered and weakly-ordered.
Most notably x86/64 provides strong ordering guarantees, while ARM
provides weak ordering guarantees. This has two consequences for concurrent
programming:

* Asking for stronger guarantees on strongly-ordered hardware may be cheap or
  even free because they already provide strong guarantees unconditionally.
  Weaker guarantees may only yield performance wins on weakly-ordered hardware.

* Asking for guarantees that are too weak on strongly-ordered hardware is
  more likely to *happen* to work, even though your program is strictly
  incorrect. If possible, concurrent algorithms should be tested on
  weakly-ordered hardware.

## Data Accesses

The C++ memory model attempts to bridge the gap by allowing us to talk about the
*causality* of our program. Generally, this is by establishing a *happens
before* relationship between parts of the program and the threads that are
running them. This gives the hardware and compiler room to optimize the program
more aggressively where a strict happens-before relationship isn't established,
but forces them to be more careful where one is established. The way we
communicate these relationships are through *data accesses* and *atomic
accesses*.

Data accesses are the bread-and-butter of the programming world. They are
fundamentally unsynchronized and compilers are free to aggressively optimize
them. In particular, data accesses are free to be reordered by the compiler on
the assumption that the program is single-threaded. The hardware is also free to
propagate the changes made in data accesses to other threads as lazily and
inconsistently as it wants. Most critically, data accesses are how data races
happen. Data accesses are very friendly to the hardware and compiler, but as
we've seen they offer *awful* semantics to try to write synchronized code with.
Actually, that's too weak.

**It is literally impossible to write correct synchronized code using only data
accesses.**

Atomic accesses are how we tell the hardware and compiler that our program is
multi-threaded. Each atomic access can be marked with an *ordering* that
specifies what kind of relationship it establishes with other accesses. In
practice, this boils down to telling the compiler and hardware certain things
they *can't* do. For the compiler, this largely revolves around re-ordering of
instructions. For the hardware, this largely revolves around how writes are
propagated to other threads. The set of orderings Rust exposes are:

* Sequentially Consistent (SeqCst)
* Release
* Acquire
* Relaxed

(Note: We explicitly do not expose the C++ *consume* ordering)

TODO: negative reasoning vs positive reasoning? TODO: "can't forget to
synchronize"

## Sequentially Consistent

Sequentially Consistent is the most powerful of all, implying the restrictions
of all other orderings. Intuitively, a sequentially consistent operation
cannot be reordered: all accesses on one thread that happen before and after a
SeqCst access stay before and after it. A data-race-free program that uses
only sequentially consistent atomics and data accesses has the very nice
property that there is a single global execution of the program's instructions
that all threads agree on. This execution is also particularly nice to reason
about: it's just an interleaving of each thread's individual executions. This
does not hold if you start using the weaker atomic orderings.

The relative developer-friendliness of sequential consistency doesn't come for
free. Even on strongly-ordered platforms sequential consistency involves
emitting memory fences.

In practice, sequential consistency is rarely necessary for program correctness.
However sequential consistency is definitely the right choice if you're not
confident about the other memory orders. Having your program run a bit slower
than it needs to is certainly better than it running incorrectly! It's also
mechanically trivial to downgrade atomic operations to have a weaker
consistency later on. Just change `SeqCst` to `Relaxed` and you're done! Of
course, proving that this transformation is *correct* is a whole other matter.

## Acquire-Release

Acquire and Release are largely intended to be paired. Their names hint at their
use case: they're perfectly suited for acquiring and releasing locks, and
ensuring that critical sections don't overlap.

Intuitively, an acquire access ensures that every access after it stays after
it. However operations that occur before an acquire are free to be reordered to
occur after it. Similarly, a release access ensures that every access before it
stays before it. However operations that occur after a release are free to be
reordered to occur before it.

When thread A releases a location in memory and then thread B subsequently
acquires *the same* location in memory, causality is established. Every write
(including non-atomic and relaxed atomic writes) that happened before A's
release will be observed by B after its acquisition. However no causality is
established with any other threads. Similarly, no causality is established
if A and B access *different* locations in memory.

Basic use of release-acquire is therefore simple: you acquire a location of
memory to begin the critical section, and then release that location to end it.
For instance, a simple spinlock might look like:

```rust
use std::sync::Arc;
use std::sync::atomic::{AtomicBool, Ordering};
use std::thread;

fn main() {
    let lock = Arc::new(AtomicBool::new(false)); // value answers "am I locked?"

    // ... distribute lock to threads somehow ...

    // Try to acquire the lock by setting it to true
    while lock.compare_and_swap(false, true, Ordering::Acquire) { }
    // broke out of the loop, so we successfully acquired the lock!

    // ... scary data accesses ...

    // ok we're done, release the lock
    lock.store(false, Ordering::Release);
}
```

On strongly-ordered platforms most accesses have release or acquire semantics,
making release and acquire often totally free. This is not the case on
weakly-ordered platforms.

## Relaxed

Relaxed accesses are the absolute weakest. They can be freely re-ordered and
provide no happens-before relationship. Still, relaxed operations are still
atomic. That is, they don't count as data accesses and any read-modify-write
operations done to them occur atomically. Relaxed operations are appropriate for
things that you definitely want to happen, but don't particularly otherwise care
about. For instance, incrementing a counter can be safely done by multiple
threads using a relaxed `fetch_add` if you're not using the counter to
synchronize any other accesses.

There's rarely a benefit in making an operation relaxed on strongly-ordered
platforms, since they usually provide release-acquire semantics anyway. However
relaxed operations can be cheaper on weakly-ordered platforms.

[C11-busted]: http://plv.mpi-sws.org/c11comp/popl15.pdf
[C++-model]: https://en.cppreference.com/w/cpp/atomic/memory_order
