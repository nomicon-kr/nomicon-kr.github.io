# 되감기

러스트는 *단계별로 나눠진* 에러 처리 계획이 있습니다:

* 만약 무언가가 합리적으로 없을 수도 있다면 `Option`이 사용됩니다.
* 만약 무언가가 잘못되었고 합리적으로 다루어질 수 있다면 `Result`가 사용됩니다.
* 만약 무언가가 잘못되었고 합리적으로 다루어질 수 없다면 스레드가 `panic!`합니다.
* 만약 어떤 파멸적인 일이 일어난다면 프로그램은 강제 종료합니다.

`Option`과 `Result`는 대부분의 상황에서 차고 넘치도록 사용되는데, 이것들이 API 사용자의 판단에 따라 `panic!`이나 강제 종료로 승격될 수 있기 때문입니다. 
`panic!`은 스레드가 정상적인 실행을 멈추고 그 스택을 되감게 만드는데, 이 때 마치 모든 함수들이 바로 반환한 것처럼 소멸자들을 호출합니다.

1.0 버전에서 러스트는 `panic!`에 대해 두 마음이 있습니다. 옛날 옛적에 러스트는 훨씬 Erlang과 같았죠. 
Erlang처럼 러스트는 경량 작업들이 있었고, 작업들은 적합하지 않은 상태가 되었을 때 `panic!`과 함께 스스로를 종료하게 설계되었습니다. Java나 C++에서의 예외와 다르게, `panic!`은 어느 때라도 처리할 수 없었습니다. 
`panic!`은 그 작업의 주인만이 처리할 수 있었죠, 물론 주인도 그 예외를 처리하거나 아니면 *스스로* `panic!`할 수도 있었습니다.

되감기는 이 이야기에서 중요했는데, 만약 어떤 작업의 소멸자가 호출되지 않는다면 메모리와 다른 시스템 자원들이 누설될 것이기 때문입니다. 
작업들은 일반적인 실행 중에 종료될 수 있었기 때문에, 러스트는 장기적인 시스템으로써는 매우 성능이 좋지 않았을 것입니다!

러스트가 현재 우리가 아는 러스트가 되어 가면서, 이런 식의 프로그래밍은 점점 추상화가 없어지면서 유행에서 뒤쳐졌습니다. 가벼운 작업들은 무거운 운영체제 스레드의 이름으로 덮여서 없어졌습니다. 
하지만 현재 1.0 러스트에서는 여전히, `panic!`은 부모 스레드에서만 처리될 수 있습니다. 이 말은 `panic!`을 처리하려면 운영체제 스레드 하나를 통째로 사용해야 한다는 겁니다! 이것은 불행하게도 러스트의 무비용 추상화의 철학에 충돌합니다.

[`catch_unwind`]라는 API가 있습니다. 이것은 스레드를 만들지 않고서 `panic!`을 처리하게 해 주죠. 하지만, 우리는 이것도 절제해서 사용하기를 권장합니다. 
좀더 자세하게 말하자면, 현재 러스트의 되감기 구현은 "되감지 않는" 경우를 위해 매우 최적화되어 있습니다. 프로그램이 되감지 않는다면, 되감기를 *준비한* 프로그램이라도 실행 시의 비용은 없을 겁니다. 
결과적으로, Java 같은 언어보다는 되감기 작업이 비용이 많이 들게 됩니다. 보통의 상황에서는 여러분의 프로그램을 되감도록 만들지 마세요. 이상적으로는, 프로그래밍 오류나 *심각한* 문제들에만 `panic!`해야 합니다.



Rust's unwinding strategy is not specified to be fundamentally compatible
with any other language's unwinding. As such, unwinding into Rust from another
language, or unwinding into another language from Rust is Undefined Behavior.
You must *absolutely* catch any panics at the FFI boundary! What you do at that
point is up to you, but *something* must be done. If you fail to do this,
at best, your application will crash and burn. At worst, your application *won't*
crash and burn, and will proceed with completely clobbered state.

[`catch_unwind`]: https://doc.rust-lang.org/std/panic/fn.catch_unwind.html
