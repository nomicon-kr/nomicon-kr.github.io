# 누설 (漏泄)

소유권 기반 자원 관리는 합성을 쉽게 하기 위한 것입니다. 객체를 만들 때 자원을 획득하고, 객체가 소멸될 때 자원을 반납합니다. 
소멸이 자동으로 처리되므로, 여러분이 자원을 반납하는 것을 까먹는 일이 없다는 뜻이고, 또한 최대한 빨리 반납이 이루어진다는 것입니다! 분명 이건 완벽하고 우리의 모든 문제는 해결된 것입니다.

모든 것은 끔찍하고 우리는 이제 새롭고 괴이한 문제들을 마주해야 합니다.

많은 사람들이 러스트가 자원 누설을 제거한다고 믿고 싶어합니다. 현실적으로, 이것은 맞는 말입니다. 안전한 러스트 프로그램이 통제되지 않는 방법으로 자원을 누설한다면 아마 놀랄 겁니다.

하지만 이론적인 측면에서 보자면 전혀 사실이 아니며, 이것은 어떻게 보든 상관없습니다. 엄밀하게 보자면, "누설"이라는 것은 너무나도 모호해서 방지할 수가 없습니다. 
어떤 컬렉션을 프로그램의 시작 때에 초기화하고, 소멸자가 있는 객체들 덩어리로 채우고, 그것을 전혀 사용하지 않는 무한 이벤트 반복문에 들어가는 것은 꽤나 흔한 일입니다. 
그 컬렉션은 쓸모 없이 앉아서 시간이나 때우겠죠, 프로그램이 종료될 때까지 귀중한 자원을 쥐고서요 (그 때에는 어차피 운영체제가 그 모든 자원들을 회수할 테니까요).

우리는 좀더 제한된 형태의 누설을 생각할 수 있습니다: 접근할 수 없는 값을 해제하는 데 실패하는 것이죠. 러스트는 이것도 막지 않습니다. 사실 러스트에는 *이것을 하는 함수가 있습니다*: `mem::forget`입니다. 
이 함수는 전달된 값을 소비하고 *그 소멸자를 실행하지 않습니다*.

예전에는 `mem::forget`이 사용되지 말라는 뜻에서 `unsafe`로 표시되었는데, 소멸자를 호출하는 데 실패하는 것은 일반적으로 좋은 행동은 아니기 때문입니다 (어떤 불안전한 코드에서는 유용해도 말이죠). 
하지만 이것은 비논리적인 행동으로 드러났습니다: 안전한 코드에서 소멸자를 호출하는 데 실패하는 방법은 엄청나게 많거든요. 가장 유명한 예제는 서로를 가리키는 `Rc<RefCell<T>>` 같은 것을 만드는 것입니다.

안전한 코드는 소멸자 누설이 발생하지 않는다고 가정하는 게 합리적인데, 소멸자를 누설하는 프로그램은 아마도 잘못된 것이기 때문입니다. 하지만 *불안전한* 코드는 안전하기 위해서 소멸자가 실행하는 것에 의지할 수는 없습니다. 
대부분의 타입은 이것과 상관이 없습니다: 만약 소멸자를 누설하면 그 타입은 정의에 의해 접근할 수 없게 되고, 따라서 별 문제가 없게 됩니다, 그렇죠? 
예를 들어, 만약 `Box<u8>`을 누설한다면 메모리를 좀 낭비하기는 하겠지만 메모리 안전성을 침해할 일은 거의 없을 겁니다.

하지만 *대리* 타입에서는 소멸자 누설을 주의해야 합니다. 이 타입들은 외부의 객체를 접근하는 것을 관리하지만, 그것을 실제로 소유하지는 않는 타입입니다. 대리 타입은 꽤 희귀합니다. 여러분이 신경써야 할 대리 객체들은 더더욱 희귀합니다.
하지만 우리는 표준 라이브러리에 있는 3개의 흥미로운 예제에 집중하겠습니다:

* `vec::Drain`
* `Rc`
* `thread::scoped::JoinGuard`

## Drain

`drain`은 컨테이너 타입을 소비하지 않고 그 컨테이너에서 데이터를 이동하는 컬렉션 API입니다. 이것을 이용하면 `Vec`의 내용물을 모두 회수한 뒤에 그 할당된 메모리를 재사용할 수 있게 되죠. 
이것은 `Vec`의 내용물을 값으로 반환하는 반복자(`Drain`)을 만들어 냅니다.

이제 `Drain`을 반복하던 도중을 생각해 봅시다: 어떤 값들은 이동되었고, 나머지는 아닙니다. 이 뜻은 `Vec`의 일부분은 이제 논리적으로 미초기화된 데이터로 가득 차 있다는 겁니다! 
우리는 값을 제거할 때마다 `Vec`의 모든 원소들을 앞으로 당길 수도 있겠지만, 그러면 꽤나 치명적인 성능 저하가 나타날 겁니다.

그 대신, 우리는 `Drain`이 해제될 때 `Vec`의 할당된 메모리를 고치는 게 좋겠습니다. 이 소멸자는 `Drain` 자신의 반복을 끝내고, 삭제되지 않은 모든 원소들을 앞으로 당기고, `Vec`의 `len`을 고칩니다. 
이것은 심지어 되감기에도 안전합니다! 쉽군요!

이제 다음의 코드를 생각해 보세요:

<!-- ignore: simplified code -->
```rust,ignore
let mut vec = vec![Box::new(0); 4];

{
    // `drain`을 시작합니다, `vec`은 더 이상 접근할 수 없습니다
    let mut drainer = vec.drain(..);

    // 2개의 원소들을 꺼내서 바로 해제시킵니다
    drainer.next();
    drainer.next();

    // `drainer`를 없애지만, 소멸자를 호출하지는 않습니다
    mem::forget(drainer);
}

// 이런, `vec[0]` 은 해제되었는데, 우리는 해제된 메모리를 가리키는 포인터를 읽고 있습니다!
println!("{}", vec[0]);
```

이건 꽤나 분명히 **좋지 않습니다**. 불행하게도, 우리는 진퇴양난의 상황에 빠졌습니다: 모든 단계에서 안정적인 상태를 유지하는 것은 엄청난 비용을 유발합니다 (그리고 API의 모든 장점을 상쇄하겠죠). 
안정적인 상태를 유지하지 못하면 안전한 코드에서 **미정의 동작이** 나올 겁니다 (API가 불건전해지겠죠).

그럼 우리는 어떻게 할까요? 음, 우리는 자명하게 안정적인 상태를 고를 수 있습니다: 반복을 시작할 때 `Vec`의 `len`을 0으로 만들고, 소멸자에서 필요하다면 `len`을 고치는 겁니다. 
이 방법이라면, 만약 모든 것이 평소처럼 동작한다면 우리는 최소의 비용으로 원하는 동작을 얻어냅니다. 하지만 만약 누군가가 반복 중간에 `mem::forget`을 사용할 *대담함이* 있다면, 그것이 초래하는 결과는 *더한 누설입니다* 
(그리고 `Vec`을 예상 밖이지만 안정적이기는 한 상태로 만듭니다). 우리는 `mem::forget`을 안전하다고 받아들였으므로, 이것은 명확하게 안전합니다. 우리는 이렇게 누설이 더한 누설을 부르는 것을 *누설 증폭이라고* 부릅니다.

## Rc

`Rc`는 흥미로운 경우인데, 처음 볼 때는 이것이 대리 타입이라고는 전혀 생각되지 않기 때문입니다. 어쨌든 이것은 가리키는 데이터를 관리하고, 모든 그 값의 `Rc`를 해제하면 그 값이 해제될 것이기 때문입니다. 
`Rc`를 누설하는 것이 그렇게 위험할 것 같지는 않은데요. 참조 횟수를 영원히 증가시킨 상태로 방치할 것이고 데이터가 해제되는 것을 막겠지만, 그건 `Box`의 경우와 같잖아요, 그렇죠?

땡, 틀렸습니다.

`Rc`의 간단한 구현을 생각해 볼까요:

<!-- ignore: simplified code -->
```rust,ignore
struct Rc<T> {
    ptr: *mut RcBox<T>,
}

struct RcBox<T> {
    data: T,
    ref_count: usize,
}

impl<T> Rc<T> {
    fn new(data: T) -> Self {
        unsafe {
            // Wouldn't it be nice if heap::allocate worked like this?
            let ptr = heap::allocate::<RcBox<T>>();
            ptr::write(ptr, RcBox {
                data,
                ref_count: 1,
            });
            Rc { ptr }
        }
    }

    fn clone(&self) -> Self {
        unsafe {
            (*self.ptr).ref_count += 1;
        }
        Rc { ptr: self.ptr }
    }
}

impl<T> Drop for Rc<T> {
    fn drop(&mut self) {
        unsafe {
            (*self.ptr).ref_count -= 1;
            if (*self.ptr).ref_count == 0 {
                // drop the data and then free it
                ptr::read(self.ptr);
                heap::deallocate(self.ptr);
            }
        }
    }
}
```

This code contains an implicit and subtle assumption: `ref_count` can fit in a
`usize`, because there can't be more than `usize::MAX` Rcs in memory. However
this itself assumes that the `ref_count` accurately reflects the number of Rcs
in memory, which we know is false with `mem::forget`. Using `mem::forget` we can
overflow the `ref_count`, and then get it down to 0 with outstanding Rcs. Then
we can happily use-after-free the inner data. Bad Bad Not Good.

This can be solved by just checking the `ref_count` and doing *something*. The
standard library's stance is to just abort, because your program has become
horribly degenerate. Also *oh my gosh* it's such a ridiculous corner case.

## thread::scoped::JoinGuard

> Note: This API has already been removed from std, for more information
> you may refer [issue #24292](https://github.com/rust-lang/rust/issues/24292).
>
> This section remains here because we think this example is still
> important, regardless of whether it is part of std or not.

The thread::scoped API intended to allow threads to be spawned that reference
data on their parent's stack without any synchronization over that data by
ensuring the parent joins the thread before any of the shared data goes out
of scope.

<!-- ignore: simplified code -->
```rust,ignore
pub fn scoped<'a, F>(f: F) -> JoinGuard<'a>
    where F: FnOnce() + Send + 'a
```

Here `f` is some closure for the other thread to execute. Saying that
`F: Send + 'a` is saying that it closes over data that lives for `'a`, and it
either owns that data or the data was Sync (implying `&data` is Send).

Because JoinGuard has a lifetime, it keeps all the data it closes over
borrowed in the parent thread. This means the JoinGuard can't outlive
the data that the other thread is working on. When the JoinGuard *does* get
dropped it blocks the parent thread, ensuring the child terminates before any
of the closed-over data goes out of scope in the parent.

Usage looked like:

<!-- ignore: simplified code -->
```rust,ignore
let mut data = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
{
    let mut guards = vec![];
    for x in &mut data {
        // Move the mutable reference into the closure, and execute
        // it on a different thread. The closure has a lifetime bound
        // by the lifetime of the mutable reference `x` we store in it.
        // The guard that is returned is in turn assigned the lifetime
        // of the closure, so it also mutably borrows `data` as `x` did.
        // This means we cannot access `data` until the guard goes away.
        let guard = thread::scoped(move || {
            *x *= 2;
        });
        // store the thread's guard for later
        guards.push(guard);
    }
    // All guards are dropped here, forcing the threads to join
    // (this thread blocks here until the others terminate).
    // Once the threads join, the borrow expires and the data becomes
    // accessible again in this thread.
}
// data is definitely mutated here.
```

In principle, this totally works! Rust's ownership system perfectly ensures it!
...except it relies on a destructor being called to be safe.

<!-- ignore: simplified code -->
```rust,ignore
let mut data = Box::new(0);
{
    let guard = thread::scoped(|| {
        // This is at best a data race. At worst, it's also a use-after-free.
        *data += 1;
    });
    // Because the guard is forgotten, expiring the loan without blocking this
    // thread.
    mem::forget(guard);
}
// So the Box is dropped here while the scoped thread may or may not be trying
// to access it.
```

Dang. Here the destructor running was pretty fundamental to the API, and it had
to be scrapped in favor of a completely different design.
