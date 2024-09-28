# 소멸자

러스트에서는 일반적인 소멸자는 없지만, 러스트에서 *제공하는* 것은 `Drop` 트레잇을 통한 완전 자동화된 소멸자입니다. 이 트레잇은 다음의 메서드를 제공합니다:

<!-- ignore: function header -->
```rust,ignore
fn drop(&mut self);
```

이 메서드는 타입에 하던 일을 끝낼 시간을 줍니다.

**`drop`이 실행된 후, 러스트는 `self`의 모든 필드들을 재귀적으로 해제하려 시도할 겁니다.**

이것은 여러분이 필드들을 해제하기 위해 "소멸자 코드 노가다"를 하지 않아도 되도록 하는 편의 기능입니다. 만약 구조체가 해제될 때 그 필드들을 해제하는 것 외에 다른 특별한 논리가 없다면, `Drop` 구현이 아예 없어도 된다는 뜻입니다!

**러스트 1.0에서는 이것을 막을 안정적인 방법은 존재하지 않습니다.**

또한 `&mut self`를 취한다는 것은 여러분이 어떻게 재귀적인 해제를 막는다고 해도, `self`에서 필드를 이동하는 등의 작업을 러스트가 막는다는 것을 의미합니다. 대부분의 타입에 있어서 이것은 전혀 문제가 없습니다.

예를 들어, `Box`의 어떤 구현은 `Drop`을 이렇게 작성할 수도 있겠습니다:

```rust
#![feature(ptr_internals, allocator_api)]

use std::alloc::{Allocator, Global, GlobalAlloc, Layout};
use std::mem;
use std::ptr::{drop_in_place, NonNull, Unique};

struct Box<T>{ ptr: Unique<T> }

impl<T> Drop for Box<T> {
    fn drop(&mut self) {
        unsafe {
            drop_in_place(self.ptr.as_ptr());
            let c: NonNull<T> = self.ptr.into();
            Global.deallocate(c.cast(), Layout::new::<T>())
        }
    }
}
fn main() {}
```

그리고 이것은 잘 작동하는데, 러스트가 `ptr` 필드를 해제하려 할 때, 실제 `Drop` 구현이 없는 [Unique]를 보기 때문입니다. 마찬가지로 `ptr`을 누구도 해제 후 사용할 수 없는데, `drop`이 종료하고 나면 접근할 수 없기 때문입니다.

하지만 다음의 코드는 동작하지 않을 겁니다:

```rust
#![feature(allocator_api, ptr_internals)]

use std::alloc::{Allocator, Global, GlobalAlloc, Layout};
use std::ptr::{drop_in_place, Unique, NonNull};
use std::mem;

struct Box<T>{ ptr: Unique<T> }

impl<T> Drop for Box<T> {
    fn drop(&mut self) {
        unsafe {
            drop_in_place(self.ptr.as_ptr());
            let c: NonNull<T> = self.ptr.into();
            Global.deallocate(c.cast(), Layout::new::<T>());
        }
    }
}

struct SuperBox<T> { my_box: Box<T> }

impl<T> Drop for SuperBox<T> {
    fn drop(&mut self) {
        unsafe {
            // 슈-퍼 최적화: `my_box`를 `drop`하지 않고 
            // 그 내용물을 해제합니다
            let c: NonNull<T> = self.my_box.ptr.into();
            Global.deallocate(c.cast::<u8>(), Layout::new::<T>());
        }
    }
}
fn main() {}
```

`SuperBox`의 소멸자에서 `my_box`의 `ptr`을 해제한 후에, 러스트는 해맑게 `my_box`에게 스스로를 해제하라고 말할 것이고, 그럼 해제 후 사용과 이중 해제로 모든 것이 폭발할 겁니다.

이러한 재귀적인 해제 동작은 `Drop`을 구현하든 하지 않든 모든 구조체와 열거형에 적용됩니다. 따라서 이러한 구조체는

```rust
struct Boxy<T> {
    data1: Box<T>,
    data2: Box<T>,
    info: u32,
}
```

그 자체로 `Drop`을 구현하지 않더라도, `data1`과 `data2`의 필드들이 해제될 "것 같을 때" 그 소멸자를 호출할 것입니다. 우리는 이런 타입이 *`Drop`이 필요하다고* 합니다, 그 자체로는 `Drop`이 아니더라도요.

마찬가지로,

```rust
enum Link {
    Next(Box<Link>),
    None,
}
```

이 열거형은 오직 그 값이 `Next`일 때만 그 안의 `Box` 필드를 해제할 것입니다.

일반적으로는 여러분이 데이터 구조를 바꿀 때마다 `drop`을 추가/삭제하지 않아도 되기 때문에 이것은 매우 좋게 동작합니다. 하지만, 여전히 소멸자를 가지고 좀더 복잡한 작업을 할 때의 많은 올바른 사용 사례가 확실히 있습니다.

재귀적 해제를 수동으로 바꿔서 `drop` 중에 `Self`에서 필드를 이동하는 것을 허용하는 고전적인 안전한 방법은 `Option`을 사용하는 것입니다:

```rust
#![feature(allocator_api, ptr_internals)]

use std::alloc::{Allocator, GlobalAlloc, Global, Layout};
use std::ptr::{drop_in_place, Unique, NonNull};
use std::mem;

struct Box<T>{ ptr: Unique<T> }

impl<T> Drop for Box<T> {
    fn drop(&mut self) {
        unsafe {
            drop_in_place(self.ptr.as_ptr());
            let c: NonNull<T> = self.ptr.into();
            Global.deallocate(c.cast(), Layout::new::<T>());
        }
    }
}

struct SuperBox<T> { my_box: Option<Box<T>> }

impl<T> Drop for SuperBox<T> {
    fn drop(&mut self) {
        unsafe {
            // 슈-퍼 최적화: `my_box`를 `drop`하지 않고
            // 그 내용물을 해제합니다. 러스트가 `my_box`를 `drop`하지 않도록
            // 이 필드를 `None`으로 설정해야 합니다.
            let my_box = self.my_box.take().unwrap();
            let c: NonNull<T> = my_box.ptr.into();
            Global.deallocate(c.cast(), Layout::new::<T>());
            mem::forget(my_box);
        }
    }
}
fn main() {}
```

하지만 이것은 좀 이상한 의미를 담고 있습니다: 항상 `Some`이어야 *하는* 필드가 `None`일 *수도 있다는* 겁니다 (소멸자에서 이런 일이 일어나기 때문에요). 
당연히 역으로는 말이 됩니다: 소멸자에서는 `self`에 임의의 메서드를 호출할 수 있고, 이렇게 하는 것이 필드를 비초기화한 후에 그 필드를 건드리지 못하게 합니다. 
하지만 이렇게 할 때 그 필드에 다른 임의의 잘못된 상태를 만드는 것은 막지 못합니다.

득과 실을 비교했을 때 이것은 괜찮은 선택입니다. 분명히 기본적으로 해야 하는 방법이죠. 하지만 미래에는 필드가 자동으로 해제되면 안된다고 말하는, 공식적인 방법이 있기를 바랍니다.

[Unique]: phantom-data.html
