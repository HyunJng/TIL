# item 30. 이왕이면 제네릭 메서드로 만들라
> 제네릭 메서드는 클래스 전체를 제네릭으로 만들 필요 없이
> 타입 안정성과 유연성을 동시에 확보할 수 있다.

## 1. 개요

메서드도 클래스처럼 **제네릭 타입 매개변수**를 가질 수 있다.
즉, 클래스 전체를 제네릭으로 만들 필요 없이,
특정 메서드에서만 타입 안정성과 재사용성을 확보할 수 있다.

---

## 2. 제네릭 메서드 기본 형태

```java
public static <T> T pick(T a, T b) {
    return a != null ? a : b;
}
```

* `<T>` : 메서드 수준에서 타입 매개변수를 정의
* `T`는 반환 타입, 매개변수 타입 등에 자유롭게 사용 가능
* 호출 시 컴파일러가 타입을 자동으로 추론한다

```java
String s = pick("a", "b");   // T는 String으로 추론
Integer i = pick(1, 2);      // T는 Integer로 추론
```

---

## 3. 제네릭 메서드의 장점

| 항목         | 설명                           |
| ---------- | ---------------------------- |
| **타입 안전성** | 명시적 캐스팅 없이 컴파일 타임 검증 가능      |
| **재사용성**   | 한 번의 구현으로 모든 타입에 사용 가능       |
| **유연성**    | 클래스 전체가 아닌, 필요한 메서드만 제네릭화 가능 |

---

## 4. 예시 1 – 재귀적 타입한정. Comparable을 이용한 제네릭 메서드

```java
public static <T extends Comparable<T>> T max(Collection<T> c) {
    if (c.isEmpty())
        throw new IllegalArgumentException("빈 컬렉션");
    T result = null;
    for (T e : c)
        if (result == null || e.compareTo(result) > 0)
            result = e;
    return result;
}
```

### (1) 설명

* `<T extends Comparable<T>>`
  → “T는 자기 자신과 비교할 수 있어야 한다”는 의미
* `compareTo()`를 통해 자연 순서를 기반으로 최대값을 비교 가능

```java
List<String> words = List.of("apple", "banana", "pear");
System.out.println(max(words)); // "pear"
```

### (2) 한계 – 하위 타입의 비교

만약 `Comparable<T>` 대신 `Comparable<? super T>`로 바꾸면 더 유연해진다.

```java
public static <T extends Comparable<? super T>> T max(Collection<T> c)
```

→ `Comparable`의 하위 타입에서도 사용 가능 

---

## 5. 예시 2 – 항등함수 (Identity Function)

책에서는 제네릭 메서드의 또 다른 활용 예로 **항등함수**를 보여준다.

```java
private static final UnaryOperator<Object> IDENTITY_FN = t -> t;

@SuppressWarnings("unchecked")
public static <T> UnaryOperator<T> identityFunction() {
    return (UnaryOperator<T>) IDENTITY_FN;
}
```

### (1) 동작 원리

* `IDENTITY_FN`은 `(Object -> Object)` 형태의 함수
* 제네릭 메서드 `<T>`를 통해 `UnaryOperator<T>`로 안전하게 변환
* 캐스팅은 안전하지만 컴파일러가 보장하지 않기 때문에 경고 억제

### (2) 사용 예시

```java
UnaryOperator<String> sameString = identityFunction();
System.out.println(sameString.apply("hello")); // hello

UnaryOperator<Integer> sameInt = identityFunction();
System.out.println(sameInt.apply(42)); // 42
```

---
## +) 항등암수 예시 추가 공부
```java
private static UnaryOperator<Object> IDENTITY_FN = (t) -> t;

@SuppressWarnings("unchecked")
public static <T> UnaryOperator<T> identityFunction() {
    return (UnaryOperator<T>) IDENTITY_FN;
}
```

---

### 1). 우선 `UnaryOperator<T>`가 뭔가?

`UnaryOperator<T>`는 **입력과 출력이 같은 타입인 함수형 인터페이스**이다.
즉, T를 받아서 T를 반환하는 함수.

```java
@FunctionalInterface
public interface UnaryOperator<T> extends Function<T, T> {
}
```

예를 들면
```java
UnaryOperator<String> toUpper = s -> s.toUpperCase();
System.out.println(toUpper.apply("hello")); // "HELLO"
```

즉, `(T) -> T` 형태를 가진 함수라고 생각하면 된다.

---

### 2) 왜 제네릭으로 만들까?

```java
UnaryOperator<Object> identity = t -> t;
```

이건 **항상 Object를 입력/출력으로 취급**하므로, 사용할 때마다 캐스팅이 필요하게 된다.

그래서 제네릭 메서드로 바꿔서, **타입 안전하게 재사용**할 수 있게 하는 것.

---

### 3) 코드 동작 원리 설명

```java
private static UnaryOperator<Object> IDENTITY_FN = (t) -> t;
```

* `IDENTITY_FN`은 실제로는 `Object -> Object` 형태의 항등함수.
* 하나만 만들어두면 어떤 타입에도 재사용할 수 있다 (stateless하기 때문).

```java
@SuppressWarnings("unchecked")
public static <T> UnaryOperator<T> identityFunction() {
    return (UnaryOperator<T>) IDENTITY_FN;
}
```

* 제네릭 메서드 `<T>`를 선언하고,
* 내부에서 `IDENTITY_FN`을 **`UnaryOperator<T>`로 캐스팅해서 반환**한다.
* 경고가 발생하므로 `@SuppressWarnings("unchecked")`를 붙인다.

---
### 4) 항등함수가 실제로 쓰이는 이유

처음 보면 “쓸모없어 보이지만”, 실무에서는 **“기본값(default function)”** 으로 자주 쓰인다.

| 상황                 | 역할                                                 |
| ------------------ | -------------------------------------------------- |
| 변환이 필요 없는 기본 변환 함수 | `transform(list, identityFunction())`              |
| Stream API         | `Collectors.toMap(keyMapper, Function.identity())` |
| 전략 패턴 기본 전략        | 조건에 따라 `Function.identity()` 사용                    |
| 함수 합성 초기값          | `Function.<T>identity()`로 안전한 시작 상태 유지             |

**즉, 항등함수는 ‘아무 일도 하지 않는 안전한 기본 함수’ 역할**을 한다.
