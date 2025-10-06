# item 11. equals를 재정의하려거든 hashCode도 재정의하라

> equals를 재정의했다면 반드시 hashCode도 재정의해야 한다.

---

## `equals`와 함께 `hashCode`도 재정의해야 하는 이유

`hashCode`는 객체를 **해시 기반 컬렉션(HashMap, HashSet, HashTable 등)** 에 저장할 때
**버킷(bucket)** 을 결정하는 데 사용된다.

만약 `equals`만 재정의하고 `hashCode`를 재정의하지 않으면,
서로 같은 객체임에도 불구하고 해시코드가 달라서 **같은 객체를 중복 저장하거나, 탐색에 실패**하는 문제가 생긴다.

예시:

```java
Map<Point, String> map = new HashMap<>();
map.put(new Point(1, 2), "A");
System.out.println(map.get(new Point(1, 2))); // null 반환
```

위 예시는 `Point`가 equals는 재정의했지만 hashCode를 재정의하지 않아,
`(1,2)` 객체를 동일하다고 판단하지 못한 사례이다.

> **버킷이란?**
> 
> HashMap은 **배열 + 연결 리스트(혹은 트리)**로 구성된 자료구조이다.
> 
>> HashMap 내부 구조 (Java 8 이후)
>> 
>> index 0 ─▶ null
>>
>> index 1 ─▶ NodeA(key1, value1) → NodeB(key2, value2)
>>
>> index 2 ─▶ null
>>
>> index 3 ─▶ NodeC(key3, value3)
> 
> 이때, 배열의 **각 칸 하나하나가 바로 “버킷(bucket)”**이다.


---

## Object의 일반 규약

> Object 명세에서 정의한 hashCode 규약은 다음과 같다.

1. **equals(Object)가 같다고 판단한 두 객체는 반드시 같은 hashCode를 반환해야 한다.**
2. **equals 비교에 사용되는 정보가 변경되지 않았다면, 애플리케이션 실행 중 hashCode는 일관된 값을 반환해야 한다.**
3. **equals(Object)가 다르다고 판단되는 두 객체라도, 같은 hashCode를 가질 수 있다.**
   (즉, 충돌은 허용된다.)

---

## 이상적인 hashCode 만드는 방법

> **좋은 해시 함수의 목표:**
>
> * 서로 다른 객체라도 **가능한 한 균등하게 해시코드가 분포**되도록 한다.
> * 연산이 빠르고 일관되어야 한다.

### 공식적인 작성 방법 (책의 권장 예시)

1. **핵심 필드 f** 들의 hashCode를 조합한다.
2. `31`을 곱해가며 누적한다.

```java
@Override
public int hashCode() {
    int result = Integer.hashCode(x);
    result = 31 * result + Integer.hashCode(y);
    return result;
}
```

### 31을 곱하는 이유

* **홀수이면서 소수(prime)** → 짝수랑 달리 1의자리까지 사용하여 해시 분포를 고르게 만든다.
* `31 * i == (i << 5) - i` 로 **비트 연산으로 최적화 가능**하다.
* 자바 컴파일러가 이를 자동으로 최적화해준다.

### 예시 (null-safe 버전)

```java
@Override
public int hashCode() {
    return Objects.hash(name, age, birthDate);
}
```

* `Objects.hash()`는 내부적으로 배열을 돌며 `31`을 곱하는 방식으로 구현되어 있다.
* 단점: varargs 배열을 매번 새로 만들어 **성능이 약간 저하될 수 있음.**
  (성능 민감하지 않은 일반 도메인 객체에는 충분히 적합함.)

---

## hashCode 지연 로딩 (lazy initialization)

`hashCode` 계산 비용이 큰 불변 객체(Immutable class)라면,
한 번 계산한 결과를 **필드에 캐싱해두고 재사용**할 수 있다.

```java
private int hashCode; // 0으로 초기화

@Override
public int hashCode() {
    int result = hashCode;
    if (result == 0) {
        result = Integer.hashCode(x);
        result = 31 * result + Integer.hashCode(y);
        hashCode = result;
    }
    return result;
}
```

> 주의:
> 이 방식은 **객체가 불변(immutable)**일 때만 안전하다.
> 가변 객체에서는 필드 값이 바뀔 때마다 hashCode가 달라질 수 있으므로 캐싱 금지.

---

## 실무에서는 오픈소스를 활용하자

현업에서는 직접 구현하기보다, **라이브러리나 어노테이션을 활용**해 실수를 줄인다.

| 방식                          | 설명                               |
| --------------------------- | -------------------------------- |
| `Objects.hash(...)`         | 표준 자바 제공, 간편하지만 느림               |
| Lombok `@EqualsAndHashCode` | equals와 hashCode를 자동 생성          |
| Google AutoValue            | 불변 객체용 코드 생성기 (컴파일 타임 생성, 성능 우수) |

### Lombok 예시

```java
@EqualsAndHashCode
public class Member {
    private String name;
    private int age;
}
```

* equals와 hashCode가 자동으로 생성된다.
* `callSuper=true` 옵션으로 상속 클래스 포함 여부도 제어 가능.