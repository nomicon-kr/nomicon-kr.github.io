# Send와 Sync

그래도 모든 것이 가변성을 물려받지는 않습니다. 어떤 타입들은 여러분이 메모리의 어떤 위치를 변경하면서도 여러 번 가리킬 수 있게 해 줍니다. 이 타입들이 이런 접근들을 동기화로 관리하지 않는다면 절대로 스레드 경계에서 안전하지 않습니다. 
러스트는 이런 특징을 `Send`와 `Sync` 트레잇을 통해 알아챕니다.

* 어떤 타입이 다른 스레드로 보내도 안전하다면 `Send`입니다.
* 어떤 타입이 스레드 간에 공유해도 안전하다면 `Sync`입니다 (`T`는 오직 `&T`가 `Send`인 경우에만 `Sync`입니다).

`Send`와 `Sync`는 러스트의 동시성 이야기에 근본이 됩니다. 그런 면에서, 이들이 잘 동작하도록 하는 상당한 양의 특수 장치들이 있습니다. 첫째로, 그리고 가장 중요한 것은, 이들은 [불안전한 트레잇입니다][unsafe_traits]. 
이 뜻은 이것들을 구현하는 것이 불안전하다는 의미이며, 다른 불안전한 코드가 이 구현들을 신뢰할 수 있다는 말입니다. 이것들이 *표시 트레잇이기* 때문에 (이것들은 메서드와 같은 연관된 것들이 없습니다), 
이것들이 잘 구현되었다는 것은 단지 구현하면 가져야 하는 본질적인 속성을 가졌다는 뜻입니다. `Send`나 `Sync`를 잘못 구현하는 것은 **미정의 동작을** 유발할 수 있습니다.

`Send`와 `Sync`는 또한 자동으로 파생되는 트레잇들입니다. 이것이 의미하는 것은, 다른 모든 트레잇과 달리, 어떤 타입이 `Send`/`Sync` 타입들로만 이루어져 있다면, 그 타입 또한 `Send`/`Sync`라는 것입니다. 
거의 모든 기본 타입이 `Send`이고 `Sync`이고, 그 결과로 여러분이 상호작용하게 될 거의 모든 타입들은 `Send`이고 `Sync`입니다. 

대표적인 예외들은 다음과 같습니다:

* 생 포인터들은 `Send`도 `Sync`도 아닙니다 (안전 장치가 없기 때문이죠).
* `UnsafeCell`은 `Sync`가 아닙니다 (그리고 따라서 `Cell`과 `RefCell`도 아닙니다).
* `Rc`는 `Send`도 `Sync`도 아닙니다 (참조 횟수가 공유되고, 동기화가 없기 때문입니다).

`Rc`와 `UnsafeCell`은 매우 근본적으로 스레드 경계에서 안전하지 않습니다: 동기화되지 않는 가변 공유 상태를 가지기 때문입니다. 하지만 생 포인터들은, 엄밀하게 말해서, *정보 차원에서* 스레드 경계 불안전하다고 표시된 것에 가깝습니다. 
생 포인터로 무언가 쓸모있는 것을 하려면 역참조를 해야 하는데, 이것은 이미 불안전하죠. 그런 점에서, 생 포인터들이 스레드 경계에서 안전하다고 표시되어도 "괜찮다"고 주장할 수 있습니다.

하지만 이것들이 스레드 경계에서 불안전하다고 표시된 것은 그들을 포함하는 타입들이 자동으로 스레드 경계에서 안전하다는 표시를 받는 것을 방지하기 위해서라는 점이 중요합니다. 
그런 타입들은 흔하지 않은, 추적되지 않는 소유권을 가지고 있는데, 그 타입의 제작자가 스레드 안전성에 대해 반드시 깊이 고민했다고 보기는 어렵습니다. 
`Rc`의 경우를 볼 때, 확실히 스레드 경계에서 안전하지 않은 `*mut`를 포함하는 타입의 좋은 예가 됩니다.

자동으로 파생되지 않는 타입들은 원할 경우 그냥 구현할 수 있습니다:

```rust
struct MyBox(*mut u8);

unsafe impl Send for MyBox {}
unsafe impl Sync for MyBox {}
```

어떤 타입이 부적절하게 자동으로 `Send`나 `Sync`로 파생되는 *굉장히 희귀한* 경우, `Send`와 `Sync`의 구현을 해제할 수도 있습니다:

```rust
#![feature(negative_impls)]

// 어떤 동기화 기초 타입을 위한 어떤 마법적인 의미가 있어요!
struct SpecialThreadToken(u8);

impl !Send for SpecialThreadToken {}
impl !Sync for SpecialThreadToken {}
```

*그 자체로는* 올바르지 않게 `Send`와 `Sync`를 파생받는 것이 불가능하다는 것에 주의하세요. 다른 불안전한 코드에 의해 특별한 의미를 부여받은 타입들만 올바르지 않게 `Send`나 `Sync`가 됨으로써 문제를 일으킬 가능성이 있습니다.

생 포인터를 사용하는 대부분의 코드는 `Send`와 `Sync`가 파생되어도 좋을 만큼 충분한 추상화 안에 갇혀야 합니다. 예를 들어 러스트의 표준 컬렉션들은 모두 `Send`와 `Sync`입니다 (`Send`와 `Sync` 타입을 포함할 때), 
비록 할당과 복잡한 소유권은 관리하기 위해 생 포인터를 많이 사용하긴 했어도 말입니다. 마찬가지로, 이 컬렉션의 대부분의 반복자들은 `Send`와 `Sync`인데, 이들이 컬렉션에 대해 많은 부분 `&`나 `&mut`와 같이 동작하기 때문입니다.

## 예제

[`Box`][box-doc]는 [여러 가지 이유 때문에][box-is-special] 컴파일러 내부의 특별한 타입으로 구현되어 있지만, 우리는 비슷한 동작을 하는 무언가를 만들어서 언제 `Send`와 `Sync`를 구현하는 것이 적절한지 예제를 삼을 수 있습니다. 
이것을 `Carton`이라 부르겠습니다.

우리는 먼저 스택에 할당된 값을 받아서 힙으로 옮기는 코드를 작성하겠습니다.

```rust
pub mod libc {
    pub use ::std::os::raw::{c_int, c_void};
    #[allow(non_camel_case_types)]
    pub type size_t = usize;
    extern "C" { pub fn posix_memalign(memptr: *mut *mut c_void, align: size_t, size: size_t) -> c_int; }
}
use std::{
    mem::{align_of, size_of},
    ptr,
    cmp::max,
};

struct Carton<T>(ptr::NonNull<T>);

impl<T> Carton<T> {
    pub fn new(value: T) -> Self {
        // 힙에 하나의 T를 저장하기에 충분한 메모리를 할당합니다.
        assert_ne!(size_of::<T>(), 0, "영량 타입은 이 예제에서는 다루지 않습니다");
        let mut memptr: *mut T = ptr::null_mut();
        unsafe {
            let ret = libc::posix_memalign(
                (&mut memptr as *mut *mut T).cast(),
                max(align_of::<T>(), size_of::<usize>()),
                size_of::<T>()
            );
            assert_eq!(ret, 0, "할당 실패 혹은 올바르지 않은 정렬선");
        };

        // NonNull은 그저 포인터가 널이 아니도록 강제하는 구조일 뿐입니다.
        let ptr = {
            // 안전성: memptr 은 우리가 레퍼런스에서 생성했고 우리가 독점적으로 가지고
            // 있기 때문에 역참조할 수 있습니다.
            ptr::NonNull::new(memptr)
                .expect("posix_memalign이 0을 반환했으니 널이 아님이 보장됨")
        };

        // 값을 스택에서 우리가 힙에 할당한 위치로 옮깁니다.
        unsafe {
            // 안전성: 널이 아니라면, posix_memalign은 유효하게 쓸 수 있고
            // 잘 정렬된 포인터를 반환합니다.
            ptr.as_ptr().write(value);
        }

        Self(ptr)
    }
}
```

이건 그렇게 쓸모가 있지는 않군요, 우리의 사용자들이 값을 주고 나면 그 값을 접근할 방법이 없네요. [`Box`][box-doc]는 [`Deref`와][deref-doc] [`DerefMut`를][deref-mut-doc] 구현해서 안의 값을 접근할 수 있게 합니다. 
우리도 이걸 합시다.

This isn't very useful, because once our users give us a value they have no way
to access it. [`Box`][box-doc] implements [`Deref`][deref-doc] and
[`DerefMut`][deref-mut-doc] so that you can access the inner value. Let's do
that.

```rust
use std::ops::{Deref, DerefMut};

struct Carton<T>(std::ptr::NonNull<T>);

impl<T> Deref for Carton<T> {
    type Target = T;

    fn deref(&self) -> &Self::Target {
        unsafe {
            // 안전성: 포인터는 [`Self::new`]의 논리에 의해 정렬되어 있고, 초기화되었으며,
            // 역참조할 수 있습니다. 우리는 이것을 읽는 사람들이 `Carton`을 빌리기를 요구하고,
            // 반환값의 수명은 입력의 수명과 동기화됩니다. 이것이 의미하는 것은 반환된 레퍼런스가
            // 해제될 때까지 아무도 `Carton`의 내용물을 변경하지 못하도록 대여 검사기가
            // 강제할 거라는 겁니다.
            self.0.as_ref()
        }
    }
}

impl<T> DerefMut for Carton<T> {
    fn deref_mut(&mut self) -> &mut Self::Target {
        unsafe {
            // 안전성: 포인터는 [`Self::new`]의 논리에 의해 정렬되어 있고, 초기화되었으며,
            // 역참조할 수 있습니다. 우리는 이것을 변경하는 사람들이 `Carton`을 가변으로 빌리기를 요구하고,
            // 반환값의 수명은 입력의 수명과 동기화됩니다. 이것이 의미하는 것은 반환된 가변 레퍼런스가
            // 해제될 때까지 아무도 `Carton`의 내용물을 접근하지 못하도록 대여 검사기가
            // 강제할 거라는 겁니다.
            self.0.as_mut()
        }
    }
}
```

마지막으로, 우리의 `Carton`이 `Send`인지, 그리고 `Sync`인지 생각해 봅시다. 어떤 타입이 가변 상태를 독점적 접근을 강제하지 않고 다른 타입과 공유하지 않으면 안전하게 `Send`가 될 수 있습니다. 
각 `Carton`은 독립된 포인터를 가지고 있으므로, 괜찮은 것 같습니다.

```rust
struct Carton<T>(std::ptr::NonNull<T>);
// 안전성: 우리 말고는 이 생 포인터를 가지고 있지 않으므로, 만약 `T`가 안전하게 다른 스레드로 보낼 수 있다면
// `Carton`도 안전하게 보낼 수 있습니다.
unsafe impl<T> Send for Carton<T> where T: Send {}
```

`Sync`는 어떨까요? `Carton`이 `Sync`가 되기 위해서 우리는 `&Carton`에 저장된 내용물이 읽히거나 변경되는 도중 다른 `&Carton`에 있는 그 내용물을 변경할 수 없다는 것을 강제해야 합니다. 
그 포인터에 쓰려면 `&mut Carton`이 필요하고, 대여 검사기가 그 가변 레퍼런스가 독점적일 것을 강제하므로, `Carton`을 `Sync`로 만드는 것에 있어서도 건전성 문제는 없습니다.

```rust
struct Carton<T>(std::ptr::NonNull<T>);
// 안전성: 
//
//
//
//
//
// Safety: Since there exists a public way to go from a `&Carton<T>` to a `&T`
// in an unsynchronized fashion (such as `Deref`), then `Carton<T>` can't be
// `Sync` if `T` isn't.
// Conversely, `Carton` itself does not use any interior mutability whatsoever:
// all the mutations are performed through an exclusive reference (`&mut`). This
// means it suffices that `T` be `Sync` for `Carton<T>` to be `Sync`:
unsafe impl<T> Sync for Carton<T> where T: Sync  {}
```

When we assert our type is Send and Sync we usually need to enforce that every
contained type is Send and Sync. When writing custom types that behave like
standard library types we can assert that we have the same requirements.
For example, the following code asserts that a Carton is Send if the same
sort of Box would be Send, which in this case is the same as saying T is Send.

```rust
struct Carton<T>(std::ptr::NonNull<T>);
unsafe impl<T> Send for Carton<T> where Box<T>: Send {}
```

Right now `Carton<T>` has a memory leak, as it never frees the memory it allocates.
Once we fix that we have a new requirement we have to ensure we meet to be Send:
we need to know `free` can be called on a pointer that was yielded by an
allocation done on another thread. We can check this is true in the docs for
[`libc::free`][libc-free-docs].

```rust
struct Carton<T>(std::ptr::NonNull<T>);
mod libc {
     pub use ::std::os::raw::c_void;
     extern "C" { pub fn free(p: *mut c_void); }
}
impl<T> Drop for Carton<T> {
    fn drop(&mut self) {
        unsafe {
            libc::free(self.0.as_ptr().cast());
        }
    }
}
```

A nice example where this does not happen is with a MutexGuard: notice how
[it is not Send][mutex-guard-not-send-docs-rs]. The implementation of MutexGuard
[uses libraries][mutex-guard-not-send-comment] that require you to ensure you
don't try to free a lock that you acquired in a different thread. If you were
able to Send a MutexGuard to another thread the destructor would run in the
thread you sent it to, violating the requirement. MutexGuard can still be Sync
because all you can send to another thread is an `&MutexGuard` and dropping a
reference does nothing.

TODO: better explain what can or can't be Send or Sync. Sufficient to appeal
only to data races?

[unsafe_traits]: safe-unsafe-meaning.html
[box-doc]: https://doc.rust-lang.org/std/boxed/struct.Box.html
[box-is-special]: https://manishearth.github.io/blog/2017/01/10/rust-tidbits-box-is-special/
[deref-doc]: https://doc.rust-lang.org/core/ops/trait.Deref.html
[deref-mut-doc]: https://doc.rust-lang.org/core/ops/trait.DerefMut.html
[mutex-guard-not-send-docs-rs]: https://doc.rust-lang.org/std/sync/struct.MutexGuard.html#impl-Send
[mutex-guard-not-send-comment]: https://github.com/rust-lang/rust/issues/23465#issuecomment-82730326
[libc-free-docs]: https://linux.die.net/man/3/free
