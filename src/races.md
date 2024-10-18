# 데이터 경합과 경합 조건

안전한 러스트는 다음과 같이 정의된 데이터 경합이 발생하지 않는 것을 보장합니다:

* 2개 이상의 스레드들이 메모리의 한 곳을 동시적으로 접근하고
* 그 중 1개 이상이 쓰기 작업을 하며
* 그 중 1개 이상이 동기화되지 않은

데이터 경합은 **미정의 동작을** 유발하므로, 안전한 러스트에서는 발생할 수 없습니다. 데이터 경합은 러스트의 소유권 규칙을 통해 *대부분* 방지됩니다: 가변 레퍼런스를 복제하는 것이 불가능하므로, 데이터 경합을 일으키는 것은 불가능합니다. 
내부 가변성은 이 문제를 더 복잡하게 만드는데, 이것이 `Send`와 `Sync` 트레잇이 있는 이유의 대부분을 차지합니다 (이것에 대해서는 다음 섹션에서 더 다룹니다).

**하지만 안전한 러스트는 일반적인 경합 조건을 방지하지 않습니다.**

이것은 스케줄러를 통제하지 않는 이상 수학적으로 불가능한데, 보통의 운영 체제 환경은 스케줄러를 통제할 수 없습니다. 프로세스 선점을 통제한다면, 일반적인 경합을 해결할 *수 있습니다* - 
이러한 기법은 [RTIC](https://github.com/rtic-rs/rtic)와 같은 프레임워크에서 사용합니다. 하지만, 스케줄링을 실제로 통제하는 것은 매우 희귀한 경우입니다.

이러한 이유 때문에, 러스트에서는 데드락에 걸리거나 올바르지 않은 동기화를 가지고 무언가 말도 안 되는 짓을 하는 것을 "안전하다"고 봅니다: 이것은 일반적인 경합 조건 혹은 자원 경합으로 알려져 있습니다. 
당연히 이런 프로그램은 좋지 않지만, 러스트가 모든 논리 오류를 잡을 수는 없기 마련입니다.

어떤 경우에서건, 경합 조건은 러스트 프로그램에서 그 자체만으로는 메모리 안전성을 침해할 수 없습니다. 반드시 어떤 다른 불안전한 코드와 엮여야만 경합 조건은 실제로 메모리 안전성을 해칠 수 있게 됩니다.
예를 들어, 올바른 프로그램은 이와 같습니다:

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
