# 데이터 경합과 경합 조건

안전한 러스트는 다음과 같이 정의된 데이터 경합이 발생하지 않는 것을 보장합니다:

* two or more threads concurrently accessing a location of memory
* one or more of them is a write
* one or more of them is unsynchronized

A data race has Undefined Behavior, and is therefore impossible to perform in
Safe Rust. Data races are prevented *mostly* through Rust's ownership system alone:
it's impossible to alias a mutable reference, so it's impossible to perform a
data race. Interior mutability makes this more complicated, which is largely why
we have the Send and Sync traits (see the next section for more on this).
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
// 우리가 실행을 마치고 나서라도 다른 스레드에서 AtomicUsize가 저장되어 있는 메모리를
// 증가시킬 수 있도록 Arc를 사용합니다. Arc가 없으면 러스트는 프로그램 컴파일을 거부할 겁니다,
// thread::spawn의 수명 요구사항 때문이죠!
let idx = Arc::new(AtomicUsize::new(0));
let other_idx = idx.clone();

// `move`는 other_idx를 값으로 흡수합니다, 이 스레드로 옮기면서 말이죠
thread::spawn(move || {
    // idx를 변경해도 됩니다, 이 값은 원자값이므로
    // 변경해도 데이터 경합을 초래할 수 없습니다.
    other_idx.fetch_add(10, Ordering::SeqCst);
});

// 원자값에서 가져온 값으로 인덱싱합니다. 이것이 안전한 이유는 우리가 원자값 메모리를 단 한 번만
// 읽고, 그 복사값을 Vec의 인덱싱 구현에 넘겨주기 때문입니다. 이 인덱싱은 똑바로 경계가 검사될
// 것이고, 중간에 값이 변경될 가능성은 없습니다. 하지만 우리의 프로그램은 이것이 실행되기 전에
// 우리가 생성한 스레드가 이 값을 증가시켰다면 panic!할 수 있습니다. 이것은 경합 조건인데, 올바른
// 프로그램 실행은 (panic!하는 것은 올바른 경우가 매우 희귀합니다) 스레드 실행의 순서에 좌우되기
// 때문입니다.
println!("{}", data[idx.load(Ordering::SeqCst)]);
```

이 코드 대신 우리가 경계 검사를 직접 하고, 데이터를 검사받지 않은 값으로 불안전하게 접근하면 데이터 경합을 일으킬 수 있습니다:
We can cause a race condition to violate memory safety if we instead do the bound
check in advance, and then unsafely access the data with an unchecked value:

```rust,no_run
use std::thread;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;

let data = vec![1, 2, 3, 4];

let idx = Arc::new(AtomicUsize::new(0));
let other_idx = idx.clone();

// `move`는 other_idx를 값으로 흡수합니다, 이 스레드로 옮기면서 말이죠
thread::spawn(move || {
    // idx를 변경해도 됩니다, 이 값은 원자값이므로
    // 변경해도 데이터 경합을 초래할 수 없습니다.
    other_idx.fetch_add(10, Ordering::SeqCst);
});

if idx.load(Ordering::SeqCst) < data.len() {
    unsafe {
        // 경계 검사를 한 후에 idx를 올바르지 않게 가져옵니다. idx는 변했을 수도 있습니다.
        // 이것은 경합 조건이고, *위험한데*, 그 이유는 우리가 `unsafe`한 `get_unchecked`를
        // 하기로 결정했기 때문입니다.
        println!("{}", data.get_unchecked(idx.load(Ordering::SeqCst)));
    }
}
```
