# 원자들 (Atomics)

러스트는 원자들의 메모리 모델에 관해서는 꽤나 대놓고 C++20의 것을 그대로 이어받습니다. 그것은 이 모델이 특별히 대단하거나 이해하기 쉬워서가 아닙니다. 
오히려 이 모델은 꽤 복잡하고 [몇 가지 문제점이 있는 것으로][C11-busted] 알려져 있습니다. 그러나 이것은 *모두가* 원자들을 설계하는 것은 잘 못한다는 사실에 실리적으로 인정하는 것입니다. 
정말 최소한, 우리는 C/C++의 메모리 모델 주변에 있는 도구들과 연구들의 혜택을 받을 수 있습니다. (여러분은 이 모델을 "C/C++11" 혹은 그냥 "C11"로 부르는 것을 볼 것입니다. 
C는 그냥 C++의 메모리 모델을 복사합니다. 또한 C++11은 그 모델의 처음 버전이었지만 그 뒤로 몇 가지 버그 수정을 받았습니다.)

이 책에서 그 모델을 전부 설명하는 것은 별로 희망이 없습니다. 실용적으로 제대로 이해하려면 하나의 책이 통째로 필요할 만큼, 사람을 정신 나가게 만드는 인과관계 그래프들로 정의되어 있거든요. 
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

한편, 비록 컴파일러가 우리가 원하는 것을 완벽히 이해하고 우리의 뜻을 존중해 주더라도, 대신 우리의 하드웨어가 우리를 문제에 빠뜨릴 수도 있습니다. 문제는 CPU 메모리 계층에 있습니다. 
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
가장 유명하기로는 x86/64 는 강하게 순서를 보장하지만, ARM은 순서의 보장이 약합니다. 이것은 동시 프로그래밍에 있어 두 가지 결과를 가져옵니다:

* 강하게 정렬된 하드웨어에서 더 강한 보장을 요구하는 것은 비용이 적거나 심지어 없습니다, 이미 무조건적으로 강한 보장을 하기 때문이죠. 약하게 정렬된 하드웨어에서 약한 보장을 요구하면 성능상의 이득만 주어질 겁니다.
* 강하게 정렬된 하드웨어에서 너무 약한 보장을 요구하면, 여러분의 프로그램이 엄밀하게는 틀렸더라도, 잘 작동하게 *될* 겁니다. 가능하다면 동시적인 알고리즘은 약하게 정렬된 하드웨어에서 테스트해야 합니다.

## 데이터 접근

C++ 메모리 모델은 우리의 프로그램에 *인과 관계에* 대해 이야기할 수 있게 해 줌으로써 이러한 차이를 줄이고자 시도합니다. 일반적으로 이것은 프로그램의 각 부분들과 그것을 실행하는 스레드들 간에 *선후* 관계를 맺음으로써 이루어집니다. 
이것은 엄밀한 선후 관계가 맺어지지 않은 부분에는 하드웨어와 컴파일러가 프로그램을 더 공격적으로 최적화할 수 있도록 하지만, 엄밀하게 선후 관계가 맺어진 곳에서는 좀더 주의하도록 강제합니다. 
우리가 이 관계들을 상호작용하는 방법은 *데이터 접근과* *원자적 접근을* 통해서입니다.

데이터 접근은 프로그래밍 세계의 기본 도구입니다. 이것은 기본적으로 동기화되지 않고, 컴파일러들은 이들을 공격적으로 최적화할 수 있습니다. 
좀더 자세하게 말하자면, 프로그램이 한 스레드만 사용한다는 가정 하에 데이터 접근은 컴파일러에 의해 순서가 재배치될 수 있습니다. 
하드웨어 또한 데이터 접근에서 일어난 변화를 다른 스레드들로 전파하는 것을 원하는 만큼 게으르고 일관성 없게 할 수 있습니다. 가장 중요하게도, 데이터 접근은 데이터 경합이 일어나는 방식입니다. 
데이터 접근은 하드웨어와 컴파일러에 매우 우호적이지만, 우리가 봤듯이 이것을 가지고 동기화된 코드를 짜려고 할 때 *끔찍한* 의미를 제공합니다.

**데이터 접근만을 가지고 올바른 동기화 코드를 작성하기는 말 그대로 불가능합니다.**

원자적 접근은 우리가 하드웨어와 컴파일러에게 우리의 프로그램이 여러 개의 스레드를 가진다고 말하는 방법입니다. 각각의 원자적 접근은 다른 접근들과 어떤 관계를 형성하는지를 특정하는 *순서로* 표시될 수 있습니다. 
실제로는 이것은 컴파일러와 하드웨어에게 할 *수 없는* 몇 가지를 말해주는 것입니다. 컴파일러에게는 이것이 명령들의 재배치에 관한 것이 대부분이고, 하드웨어에게는 쓰기 작업이 다른 스레드들로 전파되는 것에 관한 것입니다. 
러스트가 제공하는 순서들은 다음과 같습니다:

* 순서적 일관 (SeqCst)
* Release
* Acquire
* Relaxed

(주의: 우리는 C++의 *consume* 순서를 의도적으로 제공하지 않습니다)

TODO: negative reasoning vs positive reasoning? TODO: "can't forget to
synchronize"

## 순서적 일관

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
