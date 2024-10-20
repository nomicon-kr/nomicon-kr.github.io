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

컴파일러는 근본적으로 모든 종류의 복잡한 변형을 가해서 데이터 의존성을 줄이고 죽은 코드를 제거할 수 있기를 원합니다. 특히, 컴파일러는 사건들의 실제 순서를 과격하게 바꾸거나, 사건들이 아예 일어나지 않도록 할 수도 있습니다! 
우리가 만약 이런 코드를 짠다면:

<!-- ignore: simplified code -->
```rust,ignore
x = 1;
y = 3;
x = 2;
```

컴파일러는 여러분의 프로그램이 이렇게 되면 가장 좋을 것이라고 결론 내릴 수 있습니다:

<!-- ignore: simplified code -->
```rust,ignore
x = 2;
y = 3;
```

이것은 사건들의 순서를 거꾸로 만들고, 하나의 사건을 완전히 제거했습니다. 한 스레드의 관점에서 볼 때 이것은 완벽히 보이지 않습니다: 문장들을 모두 실행한 뒤에는 완전히 동일한 상태에 있으니까요. 
하지만 우리의 프로그램이 여러 개의 스레드를 가진다면, 우리는 `y`가 할당되기 전에 `x`가 1이 되는 것에 의지하고 있었을 수 있습니다. 우리는 컴파일러가 이런 최적화를 해 주었으면 좋겠습니다, 성능을 매우 향상시킬 수 있거든요. 
그렇지만, 우리는 또한 우리의 프로그램이 *우리가 말한 것을* 하도록 하기를 원합니다.

## 하드웨어 재배치

한편, 비록 컴파일러가 우리가 원하는 것을 완벽히 이해하고 우리의 뜻을 존중해 주더라도, 대신 우리의 하드웨어가 우리를 문제에 빠뜨릴 수도 있습니다. 문제는 메모리 계층의 형태로 CPU에서 옵니다. 
여러분의 하드웨어 안 어딘가에는 분명히 전역으로 공유되는 메모리 공간이 있지만, 각 CPU 코어의 입장에서 이것은 *너무 멀고* 또한 *너무나도 느립니다*. 
각 CPU는 차라리 데이터의 지역 캐시를 각자 가지고 작업하며 캐시에 그 정도 메모리가 없을 때에만 공유 메모리에 이야기하는 고통을 감수할 겁니다.

하긴 그게 캐시가 있는 이유죠, 그렇죠? 만약 캐시에서 읽는 모든 작업이 공유 메모리로 가서 캐시가 변하지 않았는지를 매번 확인해야 한다면, 캐시의 존재 이유가 뭘까요? 
최종적으로 벌어지는 일은 *한* 스레드에서 어떤 순서로 벌어지는 사건들이 *다른* 스레드에서 같은 순서로 일어나는 것을 하드웨어가 보장하지 않는다는 것입니다. 
이것을 보장하려면 우리는 CPU에게 좀 덜 똑똑하게 일을 하라는 특별한 명령들을 제공해야 합니다.

예를 들어, 우리가 컴파일러에게 이런 논리를 작성하게 했다고 합시다:

```text
처음 상태: x = 0, y = 1

스레드 1        스레드 2
y = 3;          if x == 1 {
x = 1;              y *= 2;
                }
```

이상적으로는 이 프로그램은 2가지의 가능한 최종 상태가 있습니다:

* `y = 3`: 스레드 1이 끝나기 전에 스레드 2가 상태를 확인했습니다
* `y = 6`: 스레드 1이 끝난 후에 스레드 2가 상태를 확인했습니다

그러나 여기 하드웨어가 가능할 수도 있게 하는 3번째의 상태가 있습니다:

* `y = 2`: 스레드 2는 `x = 1`을 봤지만, `y = 3`은 보지 못하고 `y = 3`을 덮어썼습니다

다른 종류의 CPU는 다른 보장들을 제공한다는 것은 한번 짚고 넘어가겠습니다. 하드웨어를 두 개의 종류로 구분하는 것은 흔한 일입니다: 강하게 정렬된 것과 약하게 정렬된 것들이죠.

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
