# repr(Rust)

첫번째로 그리고 가장 중요하게도, 모든 타입은 바이트로 표시되는 정렬선이 있습니다. 타입의 정렬선은 값을 어떤 주소에 저장하는 게 유효한지를 특정해 줍니다. `n`의 정렬선을 가지고 있는 값은 `n`의 배수인 주소에만 저장할 수 있습니다. 
따라서 정렬선이 2이면 짝수인 주소에 저장되어야 한다는 뜻이고, 1이라면 어디든지 저장될 수 있다는 뜻입니다. 정렬선은 최소 1이고, 항상 2의 거듭제곱입니다.

기본 타입들은 그들의 크기에 맞춰 정렬됩니다. 플랫폼에 따라 다르긴 하지만요. 예를 들어, x86에서는 `u64`와 `f64`는 보통 4바이트(32비트)마다 정렬됩니다.

타입의 크기는 항상 정렬선의 배수여야 합니다 (0은 어떤 정렬선이든 인정되는 크기입니다). 이것은 그 타입의 배열이 언제나 그 크기의 배수로 인덱싱되는 것을 보장합니다. 
[동량 타입][dst]의 경우에는 타입의 크기와 정렬선이 컴파일할 때 모를 수도 있다는 것을 주의하세요.

러스트는 복잡한 데이터를 다음의 방법으로 가지런하게 놓을 수 있게 해 줍니다: 

* 구조체 (곱 타입이라고 부름)
* 튜플 (이름 없는 곱 타입)
* 배열 (동형의 곱 타입)
* 열거형 (합 타입 또는 태그가 있는 공용체라고 부름)
* 공용체 (태그 없는 공용체)

열거형은 그 형(形)이 모두 연관된 데이터가 없으면 *필드가 없다*고 합니다.

기본적으로, 복합적인 자료구조는 그 필드들의 정렬선들 중 최댓값을 정렬선으로 갖습니다. 러스트는 따라서 필요한 곳에 여백을 넣음으로써 모든 필드가 잘 정렬되고, 타입의 총 크기가 그 정렬선의 배수가 되도록 합니다. 예를 들어 다음의 구조체는: 

```rust
struct A {
    a: u8,
    b: u32,
    c: u16,
}
```

이런 기본 타입들을 그들의 해당되는 크기로 정렬하는 타겟 플랫폼에서 32비트로 정렬될 것입니다. 따라서 구조체 전체는 32비트의 배수를 크기로 가지게 될 겁니다. 이 구조체는 이렇게 될 수도 있습니다: 

```rust
struct A {
    a: u8,
    _pad1: [u8; 3], // `b`를 정렬하기 위해서입니다
    b: u32,
    c: u16,
    _pad2: [u8; 2], // 전체 크기가 4바이트의 배수가 되게 하기 위해서입니다
}
```

아니면 이렇게도요: 

```rust
struct A {
    b: u32,
    c: u16,
    a: u8,
    _pad: u8,
}
```

이런 타입들에는 *간접적인 조치는 없습니다;* 모든 데이터는 C에서 그럴 것 같이, 구조체 안에 저장됩니다. 그러나 배열은 예외인데 (순서대로, 그리고 밀집되어 할당되어 있으니까요), 기본적으로 데이터의 정렬선은 특정되지 않습니다. 
다음의 두 구조체 정의를 볼 때: 

```rust
struct A {
    a: i32,
    b: u64,
}

struct B {
    a: i32,
    b: u64,
}
```

러스트는 `A` 타입의 두 값은 그 데이터가 정확히 똑같은 식으로 정렬될 것은 *보장합니다.* 그러나 러스트는, 현재로써는, `A` 타입의 값이 `B` 타입의 값과 같은 필드 순서나 여백을 가질지는 *보장하지 않습니다.*

`A`와 `B`가 이렇게 적혔으니 학술적인 느낌일 것 같지만, 러스트의 몇 가지 다른 기능들이 데이터 정렬을 여러 가지 복잡한 방법으로 가지고 놀기 좋게 해 줍니다.

예를 들어, 이 구조체를 생각해 보세요: 

```rust
struct Foo<T, U> {
    count: u16,
    data1: T,
    data2: U,
}
```
이제 이 구조체의 한 버전인 `Foo<u32, u16>`와 `Foo<u16, u32>`를 생각해 보세요. 만약 러스트가 특정된 순서로 필드를 배치한다면, 그 정렬선 문제를 해결하기 위해 구조체 안에 여백의 값을 집어넣기를 우리는 기대할 것입니다. 
따라서 만약 러스트가 필드를 재정렬하지 않았다면, 이런 식으로 출력이 나오겠지요: 

<!-- ignore: explanation code -->
```rust,ignore
struct Foo<u16, u32> {
    count: u16,
    data1: u16,
    data2: u32,
}

struct Foo<u32, u16> {
    count: u16,
    _pad1: u16,
    data1: u32,
    data2: u16,
    _pad2: u16,
}
```

후자의 경우는 꽤나 공간을 낭비합니다. 공간 사용을 최적화하려면 일반화의 다른 버전마다 *다른 필드 순서가* 필요하겠군요.

열거형은 이런 고민을 더욱 복잡하게 만듭니다. 순진하게 보자면, 이런 열거형은: 

```rust
enum Foo {
    A(u32),
    B(u64),
    C(u8),
}
```

이렇게 배치될 수 있겠습니다: 

```rust
struct FooRepr {
    data: u64, // 이것은 `tag`에 따라 u64, u32, 또는 u8 입니다.
    tag: u8,   // 0 = A, 1 = B, 2 = C
}
```

그리고 이것은 정말로 실제와 비슷하게 배치되는 것입니다 (`tag`의 위치와 크기에 따라).

하지만 어떤 몇 가지 경우에서는 이런 표현은 비효율적입니다. 전통적인 경우는 러스트의 "널 포인터 최적화"입니다: 
하나의 `()`을 가진 형(예를 들어 `None`)과 (중첩될 수도 있는) 널이 될 수 없는 포인터를 가진 형(예를 들어 `Some(&T)`)이 있는 열거형은 태그가 필요하지 않습니다. 널 포인터는 안전하게 `()` 형(`None`)으로 해석할 수 있거든요. 
최종적인 결과는, 예를 들어, 이렇게 됩니다: `size_of::<Option<&T>>() == size_of::<&T>()`

러스트에는 널이 될 수 없는 타입이나, 이를 포함하는 타입들이 많이 있는데, `Box<T>`, `Vec<T>`, `String`, `&T`, 그리고 `&mut T` 같은 것들입니다. 
비슷하게, 중첩된 열거형들이 태그를 하나의 식별자로 뭉치는 경우도 생각할 수 있는데, 그것은 그들이 정의에 의해서 유효한 값의 범위가 정해져 있기 때문입니다. 
원칙상 열거형은 꽤나 정교한 알고리즘을 써서 중첩된 타입에 있는 비트들을 금지된 값들과 함께 저장할 수 있습니다. 따라서 오늘날 우리는 열거형의 배치 상태를 밝혀지지 않은 상태로 놔두는 것이 *특별히* 좋습니다.

[dst]: exotic-sizes.html#동량動量-타입-dst
