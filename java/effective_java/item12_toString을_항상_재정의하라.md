# item 12. toString을 항상 재정의하라
> 객체를 사람이 읽기 좋은 문자열로 표현하라.

---

## `toString`을 재정의해야 하는 이유

기본적으로 `Object`의 `toString`은 다음과 같은 형태를 반환한다.

```java
getClass().getName() + "@" + Integer.toHexString(hashCode())
```

예를 들어 다음과 같다:

```java
System.out.println(new User());
```

```
User@6bc7c054
```

이 정보는 사람이 보기엔 거의 **의미가 없다.**

하지만 `toString`을 재정의하면, 디버깅·로깅·출력 시 객체의 내부 상태를 바로 이해할 수 있다.

```java
class User {
    private final String name;
    private final int age;

    public User(String name, int age) {
        this.name = name;
        this.age = age;
    }

    @Override
    public String toString() {
        return "User{name='" + name + "', age=" + age + "}";
    }
}
```

출력 결과

```
User{name='현정', age=29}
```

➡ **의미 있는 정보**를 담는 `toString`은 디버깅 효율을 극적으로 높인다.

---

## `toString` 재정의 시의 기본 원칙

### 핵심 정보만 포함하라

* 객체가 표현하는 **주요 상태**를 문자열로 보여줘야 한다.
* 너무 많은 정보(예: 내부 구현 세부사항)까지 노출하면 유지보수에 불리하다.

```java
@Override
public String toString() {
    return "Order{id=" + id + ", price=" + price + ", status=" + status + "}";
}
```

### 포맷을 문서화하라 (또는 명시적으로 “포맷이 바뀔 수 있다”고 적어라)

* 포맷을 **API로 제공**한다면 문서에 명시해야 한다.
* 예: `"name=현정, age=29"` 같은 형식은 로그 파싱 등에 쓰일 수 있으므로, 변경 시 주의 필요.
* 혹은 `"포맷은 변경될 수 있다"`라고 명시해두면 유연성을 확보할 수 있다.


### `toString`이 반환하는 문자열은 사람이 읽기 쉬워야 한다

* JSON, CSV, Key=Value 형식 등 일관된 패턴이 좋다.
* 디버깅 시 한눈에 객체의 상태를 파악할 수 있어야 한다.

---

## `toString`을 자동으로 생성하는 방법

### IDE 자동 생성

* IntelliJ / Eclipse의 “Generate → toString()” 기능 사용
  → 필요한 필드만 선택 가능

### Lombok 사용

```java
import lombok.ToString;

@ToString
public class User {
    private String name;
    private int age;
}
```

➡ 결과:

```
User(name=현정, age=27)
```

* `@ToString(exclude = "password")` 등으로 예외 지정도 가능하다.

---
## 주의사항

### `toString`에서 순환 참조 피하기

예: `Parent`가 `Child`를 포함하고, `Child`가 다시 `Parent`를 참조하면
무한루프가 발생할 수 있다.

```java
@Override
public String toString() {
    return "Parent{child=" + child + "}";
}
```

➡ `child.toString()` 내부에서 다시 `parent.toString()`을 호출할 수 있으므로 주의.