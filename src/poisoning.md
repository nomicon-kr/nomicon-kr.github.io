# 오염

모든 불안전한 코드가 최소 예외 안전성을 가지도록 *보장해야 하지만*, 모든 타입들이 *최대* 예외 안전성을 보장하는 것은 아닙니다. 어떤 타입이 최대 예외 안전성을 보장하더라도, 여러분의 코드가 그 타입에 추가적인 의미를 부여할 수 있습니다. 
예를 들어, 정수는 확실히 예외에 안전하지만, 그 자체로는 의미가 없습니다. `panic!`하는 코드가 그 정수를 올바르게 갱신하지 못할 수도 있고, 그러면 프로그램의 상태가 모순된 상태가 됩니다.

이런 것은 *보통* 괜찮은데요, 예외를 마주하는 모든 것은 곧 소멸되기 때문입니다. 예를 들어, 만약 여러분이 어떤 `Vec`을 다른 스레드로 보내고 그 스레드가 `panic!`한다면, 그 `Vec`이 이상한 상태로 있어도 상관 없습니다. 
어차피 해제되고 영원히 없어질 거니까요. 하지만 어떤 타입들은 패닉 경계를 통과해서 값들을 들여오는 것에 능합니다.

이런 타입들은 `panic!`을 마주할 경우 자신을 *오염시키도록* 명시적으로 선택할 수 있습니다. 오염은 특별히 무언가를 요구하지는 않습니다. 보통 이것이 의미하는 것은 그냥 더 이상의 정상적인 사용을 방지하는 것입니다. 
이것의 가장 주목할 만한 예는 표준 라이브러리의 `Mutex` 타입입니다. `Mutex`는 그 `MutexGuard`들 (락을 얻었을 때 `Mutex`가 반환하는 것) 중 하나가 `panic!` 중에 해제될 때, 그 자신을 오염시킵니다. 
이 이후의 `Mutex`를 잠그려는 어떤 시도도 `Err`를 반환하거나 `panic!`할 겁니다.



Mutex poisons not for true safety in the sense that Rust normally cares about. It
poisons as a safety-guard against blindly using the data that comes out of a Mutex
that has witnessed a panic while locked. The data in such a Mutex was likely in the
middle of being modified, and as such may be in an inconsistent or incomplete state.
It is important to note that one cannot violate memory safety with such a type
if it is correctly written. After all, it must be minimally exception-safe!

However if the Mutex contained, say, a BinaryHeap that does not actually have the
heap property, it's unlikely that any code that uses it will do
what the author intended. As such, the program should not proceed normally.
Still, if you're double-plus-sure that you can do *something* with the value,
the Mutex exposes a method to get the lock anyway. It *is* safe, after all.
Just maybe nonsense.
