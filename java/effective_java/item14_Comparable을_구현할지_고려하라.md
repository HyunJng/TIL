# item 14. Comparable을 구현할지 고려하라
> * `compareTo`는 **대칭성**, **추이성**, **일관성**을 반드시 보장해야 한다.
> * `equals`와 일관되게 구현하면 `TreeSet`, `TreeMap`에서 예기치 못한 동작을 방지할 수 있다.
> * Java 8 이후엔 `Comparator` 체이닝을 사용해 간결하게 작성하자.
> * “객체의 본질적인 순서”가 존재하지 않는다면 `Comparable` 대신 `Comparator`를 사용하는 것이 낫다.

## 1. Comparable 인터페이스 개념

* `Comparable` 인터페이스는 **객체 간의 자연적인 순서를 정의하는 표준 방법**을 제공한다.
* `equals`가 "같다"는 개념을 정의한다면, `compareTo`는 "**순서**"의 개념을 정의한다.
* 이 인터페이스를 구현하면 해당 클래스의 인스턴스는 `Collections.sort()`, `Arrays.sort()`, `TreeSet`, `TreeMap` 등 **정렬 가능한 컬렉션과 잘 동작**한다.

```java
public interface Comparable<T> {
    int compareTo(T o);
}
```

---

## 2. Comparable의 일반 규약

`compareTo` 메서드는 다음과 같은 규약을 지켜야 한다.

1. **대소 관계의 일관성**

    * `x.compareTo(y)` < 0 → x가 y보다 작다
    * `x.compareTo(y)` == 0 → x와 y가 같다
    * `x.compareTo(y)` > 0 → x가 y보다 크다

2. **대칭성**

    * `x.compareTo(y)`의 부호는 `y.compareTo(x)`의 부호를 반대로 해야 한다.

3. **추이성**

    * `x > y`이고 `y > z`이면 `x > z`여야 한다.

4. **equals와의 일관성**

    * `(x.compareTo(y) == 0)`이면, `x.equals(y)`도 **true**여야 한다.
      단, **필수는 아니지만 권장된다.**

        * 예: `BigDecimal("1.0")`과 `BigDecimal("1.00")`은 `compareTo`로는 같지만, `equals`로는 다르다.
        * 이런 경우, `TreeSet`이나 `TreeMap`에서는 한 객체만 저장된다.

### 1) Comparable 구현 예시

```java
public class PhoneNumber implements Comparable<PhoneNumber> {
    private final short areaCode, prefix, lineNum;

    @Override
    public int compareTo(PhoneNumber pn) {
        int result = Short.compare(areaCode, pn.areaCode);
        if (result == 0) {
            result = Short.compare(prefix, pn.prefix);
            if (result == 0) {
                result = Short.compare(lineNum, pn.lineNum);
            }
        }
        return result;
    }
}
```

## 3. 단순 비교를 위해 `Comparator`의 정적 메서드 활용 가능 (Java 8+)

```java
public class PhoneNumber implements Comparable<PhoneNumber> {
    private final short areaCode, prefix, lineNum;

    private static final Comparator<PhoneNumber> COMPARATOR =
        Comparator.comparingInt((PhoneNumber p) -> p.areaCode)
                  .thenComparingInt(p -> p.prefix)
                  .thenComparingInt(p -> p.lineNum);

    @Override
    public int compareTo(PhoneNumber pn) {
        return COMPARATOR.compare(this, pn);
    }
}
```
이 방법은 코드의 **가독성**, **유지보수성**, **안전성**이 높다.

---

## 4. compareTo 구현 시 주의사항

### 1) <, > 연산자 대신 `compare`나 `compareTo` 사용

```java
// X 잘못된 방식
return this.value < o.value ? -1 : (this.value == o.value ? 0 : 1);

// 올바른 방식
return Integer.compare(this.value, o.value);
```

* 오버플로 방지 및 명확성 확보.

---

### 2) 필드 비교 시 우선순위 명확히

* 가장 중요한 필드부터 비교하고, 같으면 다음 필드 비교.
* 예를 들어, `성 → 이름 → 나이` 순으로 정렬하려면 `thenComparing()`을 연쇄적으로 사용.

---

### 3) `compareTo`는 **타입 안정성**을 제공한다

* `Comparable`은 **제네릭**을 사용하므로, 잘못된 타입 비교 시 컴파일 오류로 잡힌다.

  ```java
  class Student implements Comparable<Student> { ... }
  ```

---

## 5. 실무에서의 활용 포인트

| 상황                                     | 추천 방법                                        | 이유                                |
| -------------------------------------- | -------------------------------------------- | --------------------------------- |
| 엔티티의 “자연스러운 순서”가 명확함 (예: 날짜, 이름, 가격 등) | `Comparable` 구현                              | `TreeSet`, `PriorityQueue` 등에서 유용 |
| 정렬 기준이 상황마다 다름                         | `Comparator` 별도로 정의                          | 하나의 자연 순서에 종속되지 않도록 분리            |
| 복합 정렬 필요                               | `Comparator.comparing()` + `thenComparing()` | 체이닝 방식으로 직관적 표현                   |

---
## 6. Comparator 사용 예시
### 1) 기본적인 단일 Comparator
```java
 Comparator<Person> byAge = Comparator.comparingInt(Person::getAge);
```
- Comparator.comparingInt : int 타입 필드를 기준으로 비교
- 내부적으로 Integer.compare()를 사용
- Person 객체를 Collections.sort()나 stream.sorted()에서 바로 사용할 수 있음

### 2) 다중 필드 비교 (기본 체이닝)
```java
 Comparator<Person> byAgeThenName =
        Comparator.comparingInt(Person::getAge)
                .thenComparing(Person::getName);
```
- thenComparing()은 첫 비교가 같을 때 두 번째 필드로 비교
- 예: 같은 나이면 이름순으로 정렬
- 가독성이 좋고, 명시적이다

### 3) 역순 정렬 (reversed)
```java
 Comparator<Person> byAgeDescThenName =
     Comparator.comparingInt(Person::getAge)
             .reversed()
             .thenComparing(Person::getName);
```
- reversed()는 결과 순서를 뒤집음

### 4) Null-safe 정렬 (nullsFirst / nullsLast)
예시: null 이름은 뒤로 보내기
```java
Comparator<Person> byNameNullLast =
      Comparator.comparing(Person::getName,
              Comparator.nullsLast(String::compareTo));
```
- Comparator.nullsFirst() / nullsLast() 로 null 안전 처리 가능
- 실무에서 DB나 외부 API 응답값에 null이 섞일 경우 매우 유용

---
## 7. Comparator의 `comparingInt()`와 `comparing()`의 차이
```java
Comparator<Person> byAge = Comparator.comparingInt(Person::getAge); // OK
Comparator<Person> byAge = Comparator.comparing(Person::getAge); // OK
```
comaringInt 메서드를 사용하지 않아도 comparing은 타입 추론이 된다.

### 1) `Comparator.comparing()`의 선언부 (JDK 17 기준)

```java
public static <T, U extends Comparable<? super U>>
Comparator<T> comparing(Function<? super T, ? extends U> keyExtractor)
```

* `keyExtractor`: 비교에 사용할 키를 추출하는 함수 (예: `Person::getAge`)
* `U`는 `Comparable`을 구현해야 함
  (`Integer`, `String`, `LocalDate`, `BigDecimal` 등 전부 Comparable)

즉,
`Person::getAge` → 리턴 타입이 `int`니까 **자동으로 `Integer`로 오토박싱** 된다.

결과적으로 내부적으로는 다음과 같이 동작한다.

```java
Comparator.comparing(Person::getAge)
== Comparator.comparing(p -> Integer.valueOf(p.getAge()))
== (p1, p2) -> Integer.compare(p1.getAge(), p2.getAge());
```

---

### 2)`comparingInt()` 선언부

```java
public static <T> Comparator<T> comparingInt(ToIntFunction<? super T> keyExtractor)
```

* `comparingInt`는 **기본형 `int` 그대로 비교**한다.
* 즉, **박싱이 일어나지 않음** → 성능상 이점이 있음
* 하지만 성능변화는 미미하므로 성능변화에 민감하지 않으면 아무거나 사용해도 상관 없다.