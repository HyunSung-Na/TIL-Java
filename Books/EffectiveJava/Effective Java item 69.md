# Effective Java item 69



### 예외는 진짜 예외 상황에만 사용하라



운이 없다면 언제가 다음과 같은 코드와 마주칠지도 모른다.



```java
// 예외를 완전히 잘못 사용한 예 - 따라 하지 말 것!

try {
    int i = 0;
    while(true)
        range[i++].climb();
} catch (ArrayIndexOutOfBoundsException e) { }
```

무슨 일을 하는 코드인지 알겠는가?  전혀 직관적이지 않다는 사실 하나만으로도 코드를 이렇게 작성하면 안 되는 이유는 충분하다. 이 코드는 배열의 원소를 순회하는데, 아주 끔찍한 방식으로 하고 있다. 무한루프를 돌다가 배열의 끝에 도달해 ArrayIndexOutOfBoundsException이 발생하면 끝을 내는 것이다. 이 코드를 다음과 같이 표준적인 관용구대로 작성했다면 모든 자바 프로그래머가 곧바로 이해했을 것이다.



```java
for (Mountain m : range)
    m.climb();
```

그런데 예외를 써서 루프를 종료한 이유는 도대체 뭘까? 잘못된 추론을 근거로 성능을 높여보려 한 것이다. JVM은 배열에 접근할 때마다 경계를 넘지 않는지 검사하는데, 일반적인 반복문도 배열 경계에 도달하면 종료한다. 따라서 이 검사를 반복문에도 명시하면 같은 일이 중복되므로 하나를 생략한 것이다. 하지만 세 가지 면에서 잘못된 추론이다.



1. 예외는 예외 상황에 쓸 용도로 설계되었으므로 JVM 구현자 입장에서는 명확한 검사만큼 빠르게 만들어야 할 동기가 약하다(최적화에 별로 신경 쓰지 않았을 가능성이 크다).
2. 코드를 try-catch 블록 안에 넣으면 JVM이 적용할 수 있는 최적화가 제한된다.
3. 배열을 순회하는 표준 관용구는 앞서 걱정한 중복 검사를 수행하지 않는다. JVM이 알아서 최적화해 없애준다.



실상은 예외를 사용한 쪽이 표준 관용구보다 훨씬 느리다.

예외를 사용한 반복문의 해악은 코드를 헷갈리게 하고 성능을 떨어뜨리는데서 끝나지 않는다. 심지어 제대로 동작하지 않을 수도 있다.



⭐ 이 야기의 교훈은 간단하다. **예외는 (그 이름이 말해주듯) 오직 예외 상황에서만 써야 한다. 절대로 일상적인 제어 흐름용으로 쓰여선 안된다.** 또한 잘 설계된 API라면 클라이언트가 정상적인 제어 흐름에서 예외를 사용할 일이 없게 해야 한다. 반복문에 예외를 사용하면 장황하고 헷갈리며 속도도 느리고, 엉뚱한 곳에서 발생한 버그를 숨기기도 한다.



- 상태 검사 메서드 대신 사용할 수 있는 선택지도 있다. 올바르지 않은 상태일 때 빈 옵셔널 혹은 null 같은 특수한 값을 반환하는 방법이다.
- 상태 검사 메서드, 옵셔널, 특정 값 중 하나를 선택하는 지침을 몇 개 소개하겠다.



1. 외부 동기화 없이 여러 스레드가 동시에 접근할 수 있거나 외부 요인으로 상태가 변할 수 있다면 옵셔널이나 특정 값을 사용한다. 상태 검사 메서드와 상태 의존적 메서드 호출 사이에 객체의 상태가 변할 수 있기 때문이다.
2. 성능이 중요한 상황에서 상태 검사 메서드가 상태 의존적 메서드의 작업 일부를 중복 수행한다면 옵셔널이나 특정 값을 선택한다.
3. 다른모든 경우엔 상태 검사 메서드 방식이 조금 더 낫다고 할 수 있다. 가독성이 살짝 더 좋고, 잘못 사용했을 때 발견하기가 쉽다. 상태 검사 메서드 호출을 깜빡 잊었다면 상태 의존적 메서드가 예외를 던져 버그를 확실히 드러낼 것이다. 반면 특정 값은 검사하지 않고 지나쳐도 발견하기가 어렵다.(옵셔널에는 해당하지 않는 문제다)



> 핵심 정리
>
> 예외는 예외 상황에서 쓸 의도로 설계되었다. 정상적인 제어 흐름에서 사용해서는 안되며, 이를 프로그래머에게 강요하는 API를 만들어서도 안된다.