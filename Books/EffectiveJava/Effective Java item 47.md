# Effective Java item 47



### 반환 타입으로는 스트림보다 컬렉션이 낫다



원소 시퀸스, 즉 일련의 원소를 반환하는 메서드는 수없이 많다. 자바 7까지는 이런 메서드의 반환 타입으로 Collection, Set, List 같은 컬렉션 인터페이스, 혹은 Iterable이나 배열을 썼다. 이 중 적합한 타입을 선택하기란 그다지 어렵지 않았다. 기본은 컬렉션 인터페이스다.

- for-each 문에서만 쓰이거나 반환된 원소 시퀸스가 일부 Collection 메서드를 구현할 수 없을 때는 Iterable 인터페이스를 썼다. 반환 원소들이 기본 타입이거나 성능에 민감한 상황이라면 배열을 썼다. 그런데 자바 8이 스트림이라는 개념을 들고 오면서 이 선택이 아주 복잡한 일이 되어버렸다.
- 스트림은 반복을 지원하지 않기 때문에 스트림과 반복을 알맞게 조합해야 좋은 코드가 나온다.
- API를 스트림만 반환하도록 짜놓으면 반환된 스트림을 for-each로 반복하길 원하는 사용자는 당연히 불만을 토로할 것이다.
- 사실 Stream 인터페이스는 Iterable 인터페이스가 정의한 추상 메서드를 전부 포함할 뿐만 아니라, Iterable 인터페이스가 정의한 방식대로 동작한다. 그럼에도 for-each로 스트림을 반복할 수 없는 까닭은 바로 Stream이 Iterable을 확장하지 않아서다. 안타깝게도 이 문제를 해결해줄 멋진 우회로는 없다.



```java
// 자바 타입 추론의 한계로 컴파일 되지 않는다.

for (ProcessHandle ph : ProcessHandle.allProcesses()::iterator) {
    // 프로세스를 처리한다.
}
```

아쉽게도 이 코드는 컴파일 오류를 낸다.

```java
Test.java:6 error: method reference not expected here
for (ProcessHandle ph : ProcessHandle.allProcesses()::iterator)    
```

이 오류를 바로잡으려면 메서드 참조를 매개변수화된 Iterable로 적절히 형변환해줘야 한다.



```java
// 스트림을 반복하기 위한 '끔찍한' 우회 방법

for (ProcessHandle ph : (Iterable<ProcessHandle>)
    					ProcessHandle.allProcesses()::iterator) {
    // 프로세스를 처리한다.
}
```

작동은 하지만 실전에 쓰기에는 너무 난잡하고 직관성이 떨어진다. 다행히 어댑터 메서드를 사용하면 상황이 나아진다. 자바는 이런 메서드를 제공하지 않지만 다음 코드와 같이 쉽게 만들어낼 수 있다. 이 경우에는 자바의 타입 추론이 문맵을 잘 파악하여 어댑터 메서드 안에서 따로 형변환하지 않아도 된다.

```java
// Stream<E>를 Iterable<E>로 중개해주는 어댑터

public static <E> Iterable<E> iterableOf(Stream<E> stream) {
    return stream::iterator;
}

// 어댑터를 사용하면 어떤 스트림도 for-each 문으로 반복할 수 있다.

for (ProcessHandle p : iterableOf(ProcessHandle.allProcesses())) {
    // 프로세스를 처리한다.
}
```

- 아이템 45의 아나그램 프로그램에서 스트림버전은 사전을 읽을 때 Files.lines 메서드를 이용했고, 반복 버전은 스캐너를 이용했다. 둘 중 파일을 읽는 동안 발생하는 모든 예외를 알아서 처리해준다는 점에서 Files.lines 쪽이 더 우수하다. 그래서 이상적으로는 반복 버전에서도 Files.lines를 써야 했다. 이는 스트림만 반환하는 API가 반환한 값을 for-each로 반복하길 원하는 프로그래머가 감수해야 할 부분이다.
- 반대로 API가 Iterable만 반환하면 이를 스트림 파이프라인에서 처리하려는 프로그래머가 성을 낸 것이다. 자바는 이를 위한 어댑터도 제공하지 않지만, 역시 손쉽게 구현할 수 있다.



```java
// Iterable<E>를 Stream<E>로 중개해주는 어댑터

public static <E> Stream<E> streamOf(Iterable<E> iterable) {
    return StreamSupport.stream(iterable.spliterator(), false);
}
```

객체 시퀸스를 반환하는 메서드를 작성하는데, 이 메서드가 오직 스트림 파이프라인에서만 쓰일 걸 안다면 마음 놓고 스트림을 반환하게 해주자. 반대로 반환된 객체들이 반복문에서만 쓰일 걸 안다면 Iterable을 반환하자.



🚀 **원소 시퀸스를 반환하는 공개 API의 반환 타입에는 Collection이나 그 하위 타입을 쓰는 게 일반적으로 최선이다.** Arrays 역시 Arrays.asList와 Stream.of 메서드로 손쉽게 반복과 스트림을 지원할 수 있다. 하지만 **단지 컬렉션을 반환한다는 이유로 덩치 큰 시퀸스를 메모리에 올려서는 안 된다.**



```java
// 입력 집합의 멱집합을 전용 컬렉션에 담아 반환한다.

public class PowerSet {
    public static final <E> Collection<Set<E>> of(Set<E> s) {
        List<E> src = new ArrayList<>(s);
        if (src.size() > 30)
            throw new IllegalArgumentException(
        			"집합에 원소가 너무 많습니다(최대 30개).: " + s);
        
        return new AbstractList<Set<E>>() {
            @Override public int size() {
                // 멱집합의 크기는 2를 원래 집합의 원소 수만큼 거듭제곱 것과 같다.
                return 1 << src.size();
            }
            
            @Override public boolean contains(Object o) {
                return o instanceof Set && src.containsAll((Set)o);
            }
            
            @Override public Set<E> get(int index) {
                Set<E> result = new HashSet<>();
                for (int i = 0; index != 0; i++, index >>= 1)
                    if ((index & 1) == 1)
                        result.add(src.get(i));
                return result;
            }
        };
    }
}
```

> 입력 집합의 원소 수가 30을 넘으면 PowerSet.of가 예외를 던진다. 이는 Stream이나 Iterable이 아닌 Collection을 반환 타입으로 쓸 때의 단점을 잘 보여준다. 다시 말해, Collection의 size 메서드가 int 값을 반환하므로 PowerSet.of가 반환되는 시퀸스의 최대 길이는 Integer.MAX_VALUE 혹은 2 ** 31 - 1로 제한된다. Collection 명세에 따르면 컬렉션이 더 크거나 심지어 무한대일 때 size가 2 ** 31 - 1을 반환해도 되지만 완전히 만족스러운 해법은 아니다.



- AbstractCollection을 활용해서 Collection 구현체를 작성할 때는 Iterable용 메서드 외에 2개만 더 구현하면 된다. 바로 contains와 size다. 이 메서드들은 손쉽게 효율적으로 구현할 수 있다. (반복이 시작되기 전에는 시퀸스의 내용을 확정할 수 없는 등의 사유로) contains와 size를 구현하는 게 불가능할 때는 컬렉션보다는 스트림이나 Iterable을 반환하는 편이 낫다. 원한다면 별도의 메서드를 두어 두 방식을 모두 제공해도 된다.
- 때로는 단순히 구현하기 쉬운 쪽을 선택하기도 한다. 예컨대 입력 리스트의 부분리스트를 모두 반환하는 메서드를 작성한다고 해보자. 필요한 부분 리스트를 만들어 표준 컬렉션에 담는 코드는 단 3줄이면 충분하다. 하지만 이 컬렉션은 입력 리스트 크기의 거듭제곱만큼 메모리를 차지한다.



```java
// 입력 리스트의 모든 부분리스트를 스트림으로 반환한다.

public class SubLists {
    public static <E> Stream<List<E>> of(List<E> list) {
        return Stream.concat(Stream.of(Collections.emptyList()),
                            prefixes(list).flatMap(SubLists::suffixes));
    }
    
    private static <E> Stream<List<E>> prefixes(List<E> list) {
        return IntStream.rangeClosed(1, list.size())
            .mapToObj(end -> list.subList(0, end));
    }
    private static <E> Stream<List<E>> suffixes(List<E> list) {
        return IntStream.range(0, list.size())
            .mapToObj(start -> list.subList(start, list.size()));
    }
}
```

Stream.concat 메서드는 반환되는 스트림에 빈 리스트를 추가하며, flatMap 메서드는 모든 프리픽스의 모든 서픽스로 구성된 하나의 스트림을 만든다. 마지막으로 프리픽스들과 서픽스들의 스트림은 IntStream.range와 IntStream.rangeClosed가 반환하는 연속된 정숫값들을 매핑해 만들었다. 쉽게 말해 이 관용구는 정수 인덱스를 사용한 표준 for 반복문의 스트림버전이라 할 수 있다. 따라서 이 구현은 for 반복문을 중첩해 만든 것과 취지가 비슷하다.



```java
for (int start = 0; start < src.size(); start++)
    for (int end = start + 1; end <= src.size(); end++)
        System.out.println(src.subList(start, end));
```

이 반복문은 그대로 스트림으로 변환할 수 있다. 그렇게 하면 앞서의 구현보다 간결해지지만, 아마도 읽기에는 더 안 좋을 것이다. 이 방식의 취지는 아이템 45에서 본 데카르트 곱용 코드와 비슷하다.



```java
// 입력 리스트의 모든 부분리스트를 스트림으로 반환한다.

public static <E> Stream<List<E>> of(List<E> list) {
    return IntStream.range(0, list.size())
        .mapToObj(start ->
                 IntStream.rangeClosed(start + 1, list.size())
                 		.mapToObj(end -> list.subList(start, end)))
        .flatMap(x -> x);
}
```

바로 앞의 for 반복문처럼 이 코드도 빈 리스트는 반환하지 않는다. 이 부분을 고치려면 앞에서처럼 concat을 사용하거나 rangeClosed 호출 코드의 1을 (int) Math.signum(start)로 고쳐주면 된다.



> 핵심 정리
>
> 원소 시퀸스를 반환하는 메서드를 작성할 때는, 이를 스트림으로 처리하기를 원하는 사용자와 반복으로 처리하길 원하는 사용자가 모두 있을 수 있음을 떠올리고, 양쪽을 다 만족시키려 노력하자. 컬렉션을 반환할 수 있다면 그렇게 하라. 반환 전부터 이미 원소들을 컬렉션에 담아 관리하고 있거나 컬렉션을 반환할 수 있다면 그렇게 하라. 반환 전부터 이미 원소들을 컬렉션에 담아 관리하고 있거나 컬렉션을 하나 더 만들어도 될 정도로 원소 개수가 적다면 ArrayList 같은 표준 컬렉션에 담아 반환하라. 그렇지 않으면 앞서의 멱집합 예처럼 전용 컬렉션을 구현할지 고민하라. 컬렉션을 반환하는 게 불가능하면 스트림과 Iterable 중 더 자연스러운 것을 반환하라. 만약 나중에 Stream 인터페이스가 Iterable을 지원하도록 자바가 수정된다면, 그때는 안심하고 스트림을 반환하면 될 것이다.(스트림처리와 반복 모두에 사용할 수 있으니)