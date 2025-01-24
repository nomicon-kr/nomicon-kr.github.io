# 구획도 (區劃圖)

먼저, 우리는 구조체의 구획도(區劃圖)를 구상해야 합니다. `Vec`은 세 개의 부분이 있습니다: 할당된 메모리를 가리키는 포인터, 할당된 메모리의 크기, 그리고 초기화된 원소들의 갯수입니다.

순진하게 보자면, 이 말은 우리가 그냥 이런 디자인을 원한다는 겁니다:

<!-- ignore: simplified code -->
```rust,ignore
pub struct Vec<T> {
    ptr: *mut T,
    cap: usize,
    len: usize,
}
```

그리고 이건 실제로 컴파일될 겁니다. 불행하게도, 이런 디자인은 너무 엄격할 겁니다. 컴파일러는 우리에게 너무 엄격한 변성을 줄 겁니다. 따라서 `&Vec<&'a str>`을 넣어야 하는 곳에 `&Vec<&'static str>`을 사용할 수는 없을 겁니다. 
변성에 관한 자세한 부분은 [소유권과 수명에 관한 챕터를][ownership] 보세요.

우리가 소유권 챕터에서 보았듯이, 자신이 소유하는 메모리를 가리키는 생 포인터를 가질 때 표준 라이브러리는 `*mut T` 대신 `Unique<T>`를 사용합니다. `Unique`는 불안정하므로, 우리는 설사 가능하더라도 이것을 사용하지 않을 겁니다.

다시 정리하면, `Unique`는 생 포인터를 감싸면서 다음의 특성들을 추가합니다:

* `T`에 대해서 공변하고
* `T` 타입의 값을 소유할 수도 있고 (이것은 우리의 예제와는 관련이 없지만, 실제 `std::vec::Vec<T>`가 이것을 필요로 하는 이유는 [`PhantomData`에 관한 챕터를][phantom-data] 보세요)
* `T`가 `Send`/`Sync`하다면 역시 `Send`/`Sync`하고
* 이 포인터는 절대 널이 아닙니다 (따라서 `Option<Vec<T>>`는 널 포인터 최적화가 됩니다)

우리는 위의 모든 요구사항들을 안정적인 러스트 버전에서 구현할 수 있습니다. 이것을 하기 위해, 우리는 `Unique<T>` 대신 [`NonNull<T>`를][NonNull] 사용할 겁니다. 이것은 생 포인터를 감싸는 또다른 구조체인데, 
위의 것들 중 두 가지 특성, 즉 `T`에 대해 공변하는 것과 절대 널이 아니라는 특성을 가집니다. `T`가 `Send`/`Sync`일 때 역시 `Send`/`Sync`하도록 구현하면, 우리는 `Unique<T>`를 사용하는 것과 같은 결과를 얻게 됩니다:

```rust
use std::ptr::NonNull;

pub struct Vec<T> {
    ptr: NonNull<T>,
    cap: usize,
    len: usize,
}

unsafe impl<T: Send> Send for Vec<T> {}
unsafe impl<T: Sync> Sync for Vec<T> {}
# fn main() {}
```

[ownership]: ../ownership.html
[phantom-data]: ../phantom-data.md
[NonNull]: https://doc.rust-lang.org/std/ptr/struct.NonNull.html
