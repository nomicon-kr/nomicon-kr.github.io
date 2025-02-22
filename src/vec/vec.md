# 실습: `Vec` 구현하기

모든 것을 아우르기 위해, 우리는 처음부터 `std::Vec`을 작성할 겁니다. 우리는 안정적인 러스트 버전에서만 작업할 것입니다. 특히 우리는 우리의 코드를 조금 더 좋거나 효율적으로 만들 수 있는 컴파일러 내장 함수들은 어떤 것도 사용하지 않을 건데, 컴파일러 내장 함수들은 항상 불안정하기 때문입니다. 비록 다른 곳들에서는 많은 내장 함수들이 *표준화되기도 하지만* 말이죠 (`std::ptr` 과 `std::mem` 은 많은 컴파일러 내장 함수로 이루어져 있습니다).

궁극적으로 이것이 의미하는 것은 우리의 구현이 가능한 모든 최적화의 혜택을 받지 않을 수도 있다는 말이지만, *순진하게 구현하겠다는 말은* 전혀 아닙니다. 우리는 당연히 껄끄러운 세부 사항들을 살펴 보고 우리의 손을 더럽힐 겁니다, 
그 문제들이 *별로* 그럴 만큼의 가치가 없을 때에도요.

고차원을 원하셨으니, 고차원으로 가겠습니다.
