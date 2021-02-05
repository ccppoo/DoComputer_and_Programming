# Multiple Dispatch: A Powerful Programming Paradigm

원본 링크 : [Multiple Dispatch: A Powerful Programming Paradigm](https://towardsdatascience.com/multiple-dispatch-a-powerful-programming-paradigm-8bc8fcd2c73a)

원본 내용은 줄리아(Julia) 프로그래밍 언어의 멀티 디스패치에 관한 내용입니다.

## How does Multiple Dispatch Work?

우선 원리에 대해서 알아보기 전에 Julia의 동적인 타입 시스템(dynamic type system)에 대해서 알아볼 필요가 있습니다.<br>
Julia의 타입은 구체(concrete)와 추상(abstract)으로 나뉩니다. 혼합(composite)과 파라미터(parametric) 형태도 있지만, 이 글에서는 다루진 않겠습니다.<br>
추상 타입은 구체 타입이 속해 있을 수 있는, 직관적으로 봤을 때 여러 구체 타입과 함게 묶여있을 수 있는 하나의 '중추'가 됩니다.

![Data Type Tree](https://miro.medium.com/max/1400/1*aZvCg36oYtsiAmk3EEWTLA.png)

Julia는 숫자를 나타낼 수 있는 방법이 다양합니다.<br>
실수, 허수, 유리수, 정수, 양수, 음수, 등 다양하게 나뉠 수 있고 숫자의 크기(int8, int16)에 따라서 나뉠 수 있습니다.<br>
그림과 같이 위계에 따라서 나뉠 수 있고, 각각의 타입은 속성에 따라 알맞은 타입의 Sub-Type에 포함됩니다.<br>
말단에 있는 'Leef node'가 구체 타입이며, 그 위로는 모두 추상타입입니다.<br>
Julia에서는 피상속을 받는 타입의 경우 무조건 추상타입입니다.<br>
이런 측면은 데이터 그 자체에 있어 상속을 받기보다는, 할 수 있는 행위(behavior)에 따라서 상속받는 것을 보여줍니다.

마지막으로 *function*과 *method*를 구분하고 넘어가겠습니다.<br>
함수(function)은 다양한 행동을 할 수 있습니다.
다양한 함수의 행동 중 하나를 메서드(method)라고 합니다.<br>

```julia
abstract type Person end
abstract type Tourist <: Person end
abstract type Deer end

encounter(a::Deer, b::Tourist) = "bows politely"
encounter(a::Tourist, b::Deer) = "feeds"
encounter(a::Person, b::Deer) = "beckons"
encounter(a::Deer, b::Person) = "ignores"
encounter(a::Tourist, b::Deer, foo::String) = "feeds deer $foo"
```

함수 *encounter*는 주어진 인자 타입에 따라  오버로드(overload) 되었으며, 주어진 인자에 따라 다르게 행동하게(behave) 됩니다.<br>
예시 코드에서 `Tourist`는 `Person`의 **서브타입이지만**, `Deer`는 `Person`과 `Tourist`와 다르게 행동합니다.<br>
정리하면, 함수가 받아들이는 인자의 종류와 순서에 따라서 다양한 행위(behavior)를 가질 수 있습니다.

인자의 타입에 따라 다르게 행동하는 것은 충분히 납득이 갈만한 상황이고, 수학에서도 불 수 있는 모습입니다.

> 멀티 디스패치는 수학 용도로 사용하기 위해서 유용합니다.
    특정 연산(operation)이 특정 인자에 속해있는 개념이 아니라는 것을 이해하실 겁니다.
    `X + Y`이라는 식에서 `+`이라는 연산자가 Y에 비해서 X를 더 중요하게 여긴다고 하면 이해가 가시나요?


## 표현성

멀티 디스패치를 구현한 언어는 그렇지 않은 다른 언어에 비해서 더 좋은 표현력을 보여줍니다.<br>
*zero dispatch*, 즉 한 함수에 인자의 개수나 타입과 상관없이 하나의 행동을 가지는 경우, 다양한 조합의 인자가 들어와도 하나의 행동 밖에 하지 못합니다.

*single dispatch*, 즉 한 함수가 하나의 인자의 타입에 따라 행위가 늘어나는 경우, 한 함수가 가질 수 있는 행위는 선형적입니다(linear).<br>
`함수 행위의 개수 = [타입의 개수] * [인자 개수]`

*multiple dispatch*, 즉 한 함수가 모든 인자의 타입과 개수에 따라 행위가 늘어나는 경우, 행위의 개수가 기하 급수적으로 늘어납니다.<br>

![](https://miro.medium.com/max/1218/1*-l--gwu9ecWiiERkDT91ig.png)

추가할 수 있는 행위의 개수만큼, 사용자 입장에서 더 추상적으로 함수(function)을 사용할 수 있고, 그에 상응하는 행위(behavior ; method)는 늘어남으로 더 구체적으로 사용할 수 있습니다.

이런 기능은 코드 재상용에 있어 활약합니다(아래에 계속).

## 코드 재사용

코드 재사용은 소프트웨어 디자인에 있어 가장 중요한 요소입니다.<br>
중복된 코드를 고치고 관리할 시간과 에너지를 최대한 줄일 수 있기 때문이죠.

멀티 디스패치가 어떻게 기존의 코드를 공유하고, 재사용하는데 크게 일조를 할까요?<br>
Julia에서는 기존에 정의된 타입에 새로운 메서드를 함수에 추가해서 기본 패키지의 재사용을 높일 수 있습니다.<br>
그러므로 코드 재사용성을 높이며, 다른 사용자가 코드를 읽는데 학습비용을 더 낮출수 있습니다.<br>
를 들어 위 예시 코드 `demo.jl` 에서 확장을 한다면, 패키지를 임포트를 한뒤에 계속해서 인자 조합에 따라 다른 행동(method)를 추가할 수 있습니다.

```julia
encounter(a::Person, b::Person) = "hello"
encounter(a::Person, b::Tourist) = "takes picture"
encounter(a::Person, b::Person, c::Person) = "converses"

dine(a::Tourist) = "picks up fork"
```

기존 함수에 인자의 조합이 다르면 다른 행동을 기대할 수 있는 점에서 직관적입니다.

사람과 관광객이 **만날 수** 있고, 사람과 사람이 **만날 수**있기 때문이죠.

Julia와 달리 정적인 언어의 경우 함수 오버로딩이 최정의 방법이 아닐 수도 있습니다.<br>
정적인 언어 중 'C++'의 경우 인자의 타입에 따라 오버로딩된 함수는 함수(function)의 행동을 결정하는 하나의 요소로 분류됩니다.<br>
런타임 중 코드가 실행되기 전까지 어떤 메서드가 실행될지 모르는 상태인 **Julia**와 달리 **C++**은 컴파일 타임 때 어떤 함수가 실행될지 결정되기 때문입니다.<br>
이런 C++의 특성은 런타임 도중 의도한 행위(behavior)를 호출하기 어렵게 만듭니다.<br>
멀티 디스패치는 Julia의 코드 재사용을 줄여주는데 큰 몫을 차지하고 있습니다.

## 요약

멀티 디스패치는 함수 인자 타입의 조합에 따라 더 높은 표현력을 가질 수 있습니다.

* 기존에 존재하는 연산(operation)에 새로운 타입이 쓸 수 있게 해줍니다.

* 기존에 존재하는 타입에 새로운 연산(operation)을 추가할 수 있습니다.

이 특성 덕분에 기존에 존재하는 패키지를 확장하기 좋고, 재사용성을 확보할 수 있습니다.<br>

멀티 디스패치는 Julia를 다른 차원의 프로그래밍언로 만드는 주요 특성중 하나입니다.