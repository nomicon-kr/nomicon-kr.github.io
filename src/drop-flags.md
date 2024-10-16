# 해제 표기

전 섹션에서의 예제들은 러스트에 흥미로운 문제를 던져줍니다. 우리는 완전히 안전하게 조건에 따라 메모리 위치를 초기화, 비초기화, 재초기화할 수 있다는 것을 봤습니다. `Copy` 타입들에게 이것은 그렇게 놀랍지 않습니다, 이 타입의 값은 그냥 비트들일 뿐이기 때문이죠. 하지만 소멸자가 있는 타입들은 이야기가 좀 다릅니다: 이 변수가 값이 할당되거나 범위에서 벗어날 때마다 소멸자를 호출해야 하는지 러스트는 알아야만 하죠. 러스트는 조건부 초기화에서 이것을 어떻게 할까요?

이것이 모든 할당이 걱정해야 하는 문제는 아니라는 것을 염두하세요. 특히, 역참조를 통해 값을 할당하는 것은 무조건적으로 해제되고, `let`으로 할당하는 것은 무조건적으로 해제되지 않습니다:

```rust
let mut x = Box::new(0); // `let`은 변수를 새로 만드므로, 해제할 필요가 없습니다
let y = &mut x;
*y = Box::new(1); // `Deref`는 피참조자가 초기화되어 있다고 가정하므로, 항상 해제됩니다
```

이것은 전에 초기화된 변수나 그 필드를 덮어쓸 때 문제인 것입니다.

사실 러스트는 타입이 해제되어야 하는지를 *실행 시간에* 추적합니다. 변수가 초기화되고 비초기화될 때, 그 변수의 *해제 표기가* 바뀝니다. 변수가 해제되어야 할 수 있을 때, 실제로 해제되어야 하는지를 결정하기 위해 이 표기를 평가합니다.

당연하게도, 프로그램의 모든 부분에서 값의 초기화 상태가 어떤지를 컴파일할 때 알 수 있는 경우가 대부분입니다. 만약 이 경우라면, 컴파일러는 이론적으로 더 효율적인 코드를 만들어낼 수 있습니다! 
예를 들어, 일직선으로 가는 코드는 이런 *정적 해제 의미*를 가지고 있습니다:

```rust
let mut x = Box::new(0); // `x`는 비초기화; 그냥 덮어씁니다.
let mut y = x;           // `y`는 비초기화; 그냥 덮어쓰고 `x`를 비초기화로 만듭니다.
x = Box::new(0);         // `x`는 비초기화; 그냥 덮어씁니다.
y = x;                   // `y`는 초기화됨; `y`를 해제하고, 덮어쓰고, `x`를 비초기화로 만듭니다!
                         // `y`가 범위 밖으로 벗어납니다; `y`는 초기화 상태였죠; `y`를 해제합니다!
                         // `x`가 범위 밖으로 벗어납니다; `x`는 비초기화 상태였죠; 아무것도 하지 않습니다.
```

비슷하게, 모든 경우가 초기화에 있어서 같은 행동을 하는 코드도 정적 해제 의미를 가집니다:

```rust
let condition = true;
let mut x = Box::new(0);    // `x`는 비초기화; 그냥 덮어씁니다.
if condition {
    drop(x)                 // `x`에서 값이 이동했습니다; `x`를 비초기화로 만듭니다.
} else {
    println!("{}", x);
    drop(x)                 // `x`에서 값이 이동했습니다; `x`를 비초기화로 만듭니다.
}
x = Box::new(0);            // `x`는 비초기화; 그냥 덮어씁니다.
                            // `x`가 범위 밖으로 벗어납니다; `x`는 초기화 상태였죠; `x`를 해제합니다!
```

하지만 다음과 같은 코드는 올바르게 해제하려면 실행 시간에 정보가 *필요합니다*:

```rust
let condition = true;
let x;
if condition {
    x = Box::new(0);        // `x`는 비초기화; 그냥 덮어씁니다.
    println!("{}", x);
}
                            // `x`가 범위 밖으로 벗어납니다; `x`는 비초기화 상태였을 수 있습니다;
                            // 표기를 확인합니다!
```

당연히 이 경우에는 정적 해제 의미를 되찾는 것이 보통입니다:

```rust
let condition = true;
if condition {
    let x = Box::new(0);
    println!("{}", x);
}
```

해제 표기는 스택에 있습니다. 예전 러스트 버전에서 해제 표기는 `Drop`을 구현하는 타입의 숨겨진 필드 안에 있었습니다.
