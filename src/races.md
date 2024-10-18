# 데이터 경합과 경합 조건

안전한 러스트는 다음과 같이 정의된 데이터 경합이 발생하지 않는 것을 보장합니다:

* 2개 이상의 스레드들이 메모리의 한 곳을 동시적으로 접근하고
* 그 중 1개 이상이 쓰기 작업을 하며
* 그 중 1개 이상이 동기화되지 않은

데이터 경합은 **미정의 동작을** 유발하므로, 안전한 러스트에서는 발생할 수 없습니다. 데이터 경합은 러스트의 소유권 규칙을 통해 *대부분* 방지됩니다: 가변 레퍼런스를 복제하는 것이 불가능하므로, 데이터 경합을 일으키는 것은 불가능합니다. 
내부 가변성은 이 문제를 더 복잡하게 만드는데, 이것이 `Send`와 `Sync` 트레잇이 있는 이유의 대부분을 차지합니다 (이것에 대해서는 다음 섹션에서 더 다룹니다).

**하지만 안전한 러스트는 일반적인 경합 조건을 방지하지 않습니다.**

This is mathematically impossible in situations where you do not control the
scheduler, which is true for the normal OS environment. If you do control
preemption, it _can be_ possible to prevent general races - this technique is
used by frameworks such as [RTIC](https://github.com/rtic-rs/rtic). However,
actually having control over scheduling is a very uncommon case.

For this reason, it is considered "safe" for Rust to get deadlocked or do
something nonsensical with incorrect synchronization: this is known as a general
race condition or resource race. Obviously such a program isn't very good, but
Rust of course cannot prevent all logic errors.

In any case, a race condition cannot violate memory safety in a Rust program on
its own. Only in conjunction with some other unsafe code can a race condition
actually violate memory safety. For instance, a correct program looks like this:

```rust,no_run
use std::thread;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;

let data = vec![1, 2, 3, 4];
// Arc so that the memory the AtomicUsize is stored in still exists for
// the other thread to increment, even if we completely finish executing
// before it. Rust won't compile the program without it, because of the
// lifetime requirements of thread::spawn!
let idx = Arc::new(AtomicUsize::new(0));
let other_idx = idx.clone();

// `move` captures other_idx by-value, moving it into this thread
thread::spawn(move || {
    // It's ok to mutate idx because this value
    // is an atomic, so it can't cause a Data Race.
    other_idx.fetch_add(10, Ordering::SeqCst);
});

// Index with the value loaded from the atomic. This is safe because we
// read the atomic memory only once, and then pass a copy of that value
// to the Vec's indexing implementation. This indexing will be correctly
// bounds checked, and there's no chance of the value getting changed
// in the middle. However our program may panic if the thread we spawned
// managed to increment before this ran. A race condition because correct
// program execution (panicking is rarely correct) depends on order of
// thread execution.
println!("{}", data[idx.load(Ordering::SeqCst)]);
```

We can cause a data race if we instead do the bound check in advance, and then
unsafely access the data with an unchecked value:

```rust,no_run
use std::thread;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;

let data = vec![1, 2, 3, 4];

let idx = Arc::new(AtomicUsize::new(0));
let other_idx = idx.clone();

// `move` captures other_idx by-value, moving it into this thread
thread::spawn(move || {
    // It's ok to mutate idx because this value
    // is an atomic, so it can't cause a Data Race.
    other_idx.fetch_add(10, Ordering::SeqCst);
});

if idx.load(Ordering::SeqCst) < data.len() {
    unsafe {
        // Incorrectly loading the idx after we did the bounds check.
        // It could have changed. This is a race condition, *and dangerous*
        // because we decided to do `get_unchecked`, which is `unsafe`.
        println!("{}", data.get_unchecked(idx.load(Ordering::SeqCst)));
    }
}
```
