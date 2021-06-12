# Effective Java item 86



### Serializable을 구현할지는 신중히 결정하라



어떤 클래스의 인스턴스를 직렬화할 수 있게 하려면 클래스 선언에 implements Serializable만 덧붙이면 된다. 너무 쉽게 적용할 수 있기 때문에 프로그래머가 특별히 신경 쓸 게 없다는 오해가 생길 수 있지만, 진실은 훨씬 더 복잡하다. 짧게 보면 손쉬워 보이지만, 길게 보면 아주 값비싼 일이다.



- **Serializable을 구현하면 릴리스한 뒤에는 수정하기 어렵다.** 클래스가 Serializable을 구현하면 직렬화된 바이트 스트림 인코딩도 하나의 공개 API가 된다. 그래서 이 클래스가 널리 퍼진다면 그 직렬화 형태도 영원히 지원해야 하는 것이다.
- 기본 직렬화 형태에서는 클래스의 private와 package-private 인스턴스 필드들마저 API로 공개되는 꼴이 된다.(캡슐화가 깨진다.)
- 뒤늦게 클래스 내부 구현을 손보면 원래의 직렬화 형태와 달라지게 된다. 한쪽은 구버전 인스턴스를 직렬화하고 다른쪽은 신버전 클래스로 역직렬화한다면 실패를 맛볼 것이다.



### 문제들



1. 모든 직렬화된 클래스는 고유 식별 번호를 부여 받는다. serialVersionUID라는 이름의 static final long 필드로, 이 번호를 명시하지 않으면 시스템이 런타임에 암호 해시 함수를 적용해 자동으로 클래스 안에 생성해 넣는다. 이 값을 생성하는 데는 클래스 이름, 구현한 인터페이스들, 컴파일러가 자동으로 생성해 넣은 것을 포함한 대부분의 클래스 멤버들이 고려된다. 나중에 이들 중 하나라도 수정한다면 UID 값도 변한다. **다시 말해 자동 생성되는 값에 의존하면 쉽게 호환성이 깨져버려 런타임에 InvalidClassException이 발생할 것이다.**
2. **두 번째 문제는 버그와 보안 구멍이 생길 위험이 높아진다.** 객체는 생성자를 사용해 만드는 게 기본이다. 즉 직렬화는 언어의 기본 메커니즘을 우회하는 객체 생성 기법인 것이다.
3. **세번째 문제는 해당 클래스의 신버전을 릴리스할 때 테스트할 것이 늘어난다는 점이다.** 직렬화 가능 클래스가 수정되면 신버전 인스턴스를 직렬화한 후 구버전으로 역직렬화할 수 있는지, 그리고 그 반대도 가능한지를 검사해야 한다. 따라서 테스트해야 할 양이 직렬화 가능 클래스의 수와 릴리스 횟수에 비례해 증가한다.



- **Serializable 구현 여부는 가볍게 결정할 사안이 아니다.** 단 객체를 전송하거나 저장할 때 자바 직렬화를 이용하는 프레임워크용으로 만든 클래스라면 선택의 여지가 없다.역사적으로 BigInteger와 Instant 같은 '값' 클래스와 컬렉션 클래스들은 Serializable을 구현하고, 스레드 풀처럼 '동작'하는 객체를 표현하는 클래스들은 구현하지 않았다.
- **상속용으로 설계된 클래스는 대부분 Serializable을 구현하면 안 되며, 인터페이스도 대부분 Serializable을 확장해서는 안 된다.** 이 규칙을 따르지 않으면, 그런 클래스를 확장하거나 그런 인터페이스를 구현하는 이에게 커다란 부담을 지우게 된다.
- 상속용으로 설계된 클래스 중 Serializable을 구현한 예로는 Throwable과 Component가 있다. Throwable은 서버가 RMI를 통해 클라이언트로 예외를 보내기 위해 구현했다. Component는 GUI를 전송하고 저장하고 복원하기 위해 구현하였다.



✔ 만약 여러분이 작성하는 클래스의 인스턴스 필드가 직렬화와 확장이 모두 가능하다면 주의할 점이 몇 가지 있다. 인스턴스 필드 값 중 불변식을 보장해야 할 게 있다면 반드시 하위 클래스에서 finalize 메서드를 재정의하지 못하게 해야 한다.



✔ 마지막으로, 인스턴스 필드 중 기본 값으로 초기화되면 위배되는 불변식이 있다면 클래스에 다음의 readObjectNoData 메서드를 반드시 추가해야 한다.

```java
// 상태가 있고, 확장가능하고, 직렬화 가능한 클래스용 readObjectNoData 메서드

private void readObjectNoData() throws InvalidObjectException {
    throw new InvalidObjectException("스트림 데이터가 필요합니다.");
}
```

이 메서드는 자바 4에 추가된 것으로, 기존의 직렬화 가능 클래스에 직렬화 가능 상위 클래스를 추가하는 드문 경우를 위한 메서드다.



**내부 클래스는 직렬화를 구현하지 말아야 한다.** 내부 클래스에는 바깥 인스턴스의 참조와 유효 범위 안의 지역변수 값들을 저장하기 위해 컴파일러가 생성한 필드들이 자동으로 추가된다. 이 필드들이 클래스 정의에 어떻게 추가되는지도 정의되지 않았다. 다시 말해 내부 클래스에 대한 기본 직렬화 형태는 분명하지가 않다.



> 핵심 정리
>
> Serializable은 구현한다고 선언하기는 아주 쉽지만. 그것은 눈속임일 뿐이다. 한 클래스의 여러 버전이 상호작용할 일이 없고 서버가 신뢰할 수 없는 데이터에 노출될 가능성이 없는 등, 보호된 환경에서만 쓰일 클래스가 아니라면 Serializable 구현은 아주 신중하게 이뤄져야 한다. 상속할 수 있는 클래스라면 주의사항이 더욱 많아진다.