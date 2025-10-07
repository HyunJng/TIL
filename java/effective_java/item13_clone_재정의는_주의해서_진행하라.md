# item 13. `clone` 재정의는 주의해서 진행하라
> `Cloneable`과 `clone()`은 설계 결함이 많은 API이다.
> 불변식이 깨지고, 깊은 복사를 강제할 수 없으며, 상속 구조에서 특히 취약하다.
>
> 따라서 **새로운 코드에서는 `clone()`을 피하고**,
> **복사 생성자나 복사 팩터리**를 사용하는 것이 이펙티브 자바가 권장하는 방식이다.
> (배열은 `clone()`을 사용 해도 좋다)
---

## `clone()` 메서드의 개념

모든 클래스는 `Object`를 상속받고, `Object`에는 다음과 같은 대표적인 메서드들이 있다.

```java
equals(), hashCode(), toString(), clone(), finalize()
```

`clone()`은 **객체를 복제하는 용도**로 설계된 protected 메서드이다.

```java
protected native Object clone() throws CloneNotSupportedException;
```

* 기본 구현은 **해당 객체의 필드 값을 그대로 복사(얕은 복사)** 한다.
* 즉, 기본형 필드는 안전하지만, **참조형 필드는 주소값만 복사되어 원본과 같은 객체를 참조**하게 된다.

---

## 얕은 복사 vs 깊은 복사

| 구분                   | 설명                               | 예시                                   |
| -------------------- | -------------------------------- | ------------------------------------ |
| 얕은 복사 (Shallow Copy) | 객체의 필드 값만 그대로 복사 (참조 타입은 주소값 복사) | `Object.clone()`의 기본 동작              |
| 깊은 복사 (Deep Copy)    | 내부 참조 객체까지 새로 만들어 복제             | 내부 필드도 직접 `clone()` 또는 `new`로 복제해야 함 |

```java
class Person implements Cloneable {
    String name;
    Address address; // 참조 타입

    @Override
    protected Person clone() throws CloneNotSupportedException {
        Person cloned = (Person) super.clone(); // 얕은 복사
        cloned.address = address.clone();        // 깊은 복사
        return cloned;
    }
}
```

참조 필드까지 복제하지 않으면, 원본과 복제본이 같은 객체를 바라보기 때문에
**한쪽의 변경이 다른 쪽에도 영향을 준다.**

---

## `Cloneable` 인터페이스의 역할

* `Cloneable`은 **마커 인터페이스(marker interface)** — 메서드가 전혀 없다.
* 단지 “이 객체는 복제를 허용한다”는 **신호 역할**을 한다.
* 구현하지 않은 객체에서 `Object.clone()`을 호출하면 `CloneNotSupportedException`이 발생한다.

```java
class Person implements Cloneable {
    @Override
    protected Object clone() throws CloneNotSupportedException {
        return super.clone(); // Object.clone() 호출
    }
}
```

---

## `=` 과 `Object.clone()`의 차이

| 구분               | 설명                            |
| ---------------- | ----------------------------- |
| `=` (대입 연산)      | 같은 객체를 가리킨다. 즉, 참조 주소만 복사한다.  |
| `Object.clone()` | 새로운 객체를 생성하고, 필드 값을 그대로 복사한다. |

```java
Person origin = new Person("Alice");
Person alias = origin;          // 같은 객체를 가리킴
Person copied = origin.clone(); // 완전히 새로운 객체

System.out.println(origin == alias); // true
System.out.println(origin == copied); // false
```

즉, `=`은 **참조 복사**, `clone()`은 **객체 복사(단, 얕은 복사)** 이다.

---

## `clone()`의 문제점

### 1. 얕은 복사의 위험성

```java
class Address {
    String city;
    Address(String city) { this.city = city; }
}

class Person implements Cloneable {
    String name;
    Address address;

    Person(String name, String city) {
        this.name = name;
        this.address = new Address(city);
    }

    @Override
    protected Person clone() throws CloneNotSupportedException {
        return (Person) super.clone(); // 얕은 복사
    }
}
```

```java
Person p1 = new Person("Alice", "Seoul");
Person p2 = p1.clone();

p2.name = "Bob";           // p1.name은 그대로 "Alice"
p2.address.city = "Busan"; // p1.address.city도 "Busan"으로 변경됨
```

* `clone()`은 **“이 객체(Person)”** 만 새로 만들 뿐,
  내부의 **참조 필드(Address)** 는 그대로 복사한다.
* 즉, **내부 가변 객체(mutable object)** 를 공유하게 되어 의도치 않은 부작용이 생긴다.

---

### 2. `Cloneable`은 단순 신호만 제공하고, 안전장치가 없다

* `Cloneable`은 “복제 가능하다는 의사표시”일 뿐, 복제 **규약을 enforce하지 않는다.**
* 하위 클래스에서 `clone()`을 잘못 재정의해도 컴파일러가 막아주지 않는다.

---

### 3. 생성자를 호출하지 않는다 → 불변식(invariant)이 깨질 수 있다

* `Object.clone()`은 **생성자를 거치지 않기 때문에**,
  클래스 내부의 불변 조건이나 검증 로직이 건너뛰어진다.

예: ID 생성, 로그 등록, 락 획득, 리소스 핸들링 같은 초기화 로직이 전혀 수행되지 않는다.

---

### 4️. 상속 구조에서 깨지기 쉽다

- 부모 클래스가 `Cloneable`을 구현하지 않았다면, 하위 클래스의 `clone()`도 동작하지 않는다.
- 상속 구조가 깊을수록 `super.clone()` 체인이 깨질 위험이 높다.
- 따라서 **상속 구조에서는 `clone()`보다 복사 생성자나 복사 팩터리로 복제를 구현하는 게 훨씬 안전하다.**

---

## 안전하게 `clone()`을 재정의하는 패턴

```java
@Override
protected Person clone() {
    try {
        Person copy = (Person) super.clone();      // 1. 얕은 복사
        copy.address = new Address(address.city);  // 2. 가변 필드는 새로 복제
        return copy;
    } catch (CloneNotSupportedException e) {
        throw new AssertionError(); // 발생할 일 없음
    }
}
```

* `CloneNotSupportedException`은 checked이지만,
  `Cloneable`을 이미 구현했기 때문에 절대 발생하지 않는다.
  → 따라서 `AssertionError`로 감싸는 것이 관용적 패턴이다.

---

## `clone()` 대신 복사 생성자 / 복사 팩터리 사용 권장
> 단, 배열을 복제할 떄는 `clone()`을 사용해도 좋다!
### 복사 생성자 (Copy Constructor)

```java
public Person(Person original) {
    this.name = original.name;
    this.address = new Address(original.address.city);
}
```

### 복사 팩터리 (Copy Factory)

```java
public static Person newInstance(Person original) {
    return new Person(original);
}
```

이 방식은 다음과 같은 장점이 있다

* 생성자를 거치므로 **불변식 보장**
* **final 필드 복사** 가능
* 예외 설계 자유로움 (`throws` 불필요)
* **제네릭 / 상속 구조**에서도 안정적
* API 명세가 명확함 (`clone()`처럼 숨겨진 규약이 없음)
---
## ➕ 책보면서 추가 공부한 것
### 1. 배열에서 clone()이 문제가 없으면 Collection 도 상관 없을까? ` NO`
“참조 타입 배열”과 “컬렉션”이 모두 ‘요소 참조를 얕게 복사한다’는 점에서는 비슷하지만, 
배열의 `clone()`과 달리 컬렉션의 `clone()`은 권장되지 않는다.

* **언어 차원의 일관성(배열) vs 구현체별 제각각(컬렉션)**

    * 배열: 모든 배열 타입이 JLS에 따라 `public Object clone()`을 갖고, 동작이 항상 동일(새 배열 인스턴스 + 요소 참조의 얕은 복사).
    * 컬렉션: `Collection`/`Map` 인터페이스에 `clone()`이 없다. 구현체마다 유무/반환타입/세부동작이 제각각이고, 아예 `Cloneable`을 지원하지 않는 래퍼(`Collections.unmodifiable*`)도 있다. 

* **API 명세의 명확성**

    * 배열: “새 배열 생성 + 요소 참조 복사”가 **명세로 보장**된다.
    * 컬렉션: `ArrayList.clone()`, `HashMap.clone()` 등은 대체로 얕은 복사지만, 일관된 인터페이스 규약이 없습니다. 실수/오해 여지가 큽다.

* **타입(제네릭) 안정성**

    * 배열: 런타임에 요소 타입 정보가 살아있는 재화형(reifiable) 이다. `clone()`이 동일한 요소 타입의 새 배열을 반환함이 보장됩니다.
    * 컬렉션: 제네릭은 타입 소거가 일어나고, `clone()` 반환 타입도 `Object`라서 결국 캐스팅해야 한다.

* **의도 전달과 유지보수성**

    * 배열: `arr.clone()`만 보면 “백킹 스토리지만 분리”하려는 의도가 명확하다. Stack 예제처럼 버퍼 분리 용도로 이상적이다.
    * 컬렉션: `clone()`은 표준적 패턴이 아니라서, 팀 차원에서 의도를 읽기 어렵고 오해 소지가 있다.

### 2. "clone메서드 역시 적절히 동기화해줘야 한다" 는 말이 무슨 뜻일까?
`clone()`은 단순히 메모리를 복사하는 동작이다. 그런데 복사 도중에 다른 스레드가 원본 객체를 수정하고 있다면?

→ **복제본의 일관성이 깨질 수 있다.**

**예시**

```java
class Counter implements Cloneable {
    private int count;

    public void increment() {
        count++;
    }

    @Override
    protected Counter clone() {
        try {
            return (Counter) super.clone(); // 얕은 복사
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }
}
```

만약 한 스레드가 `count`를 증가시키는 중(`increment`)이고,
다른 스레드가 동시에 `clone()`을 호출하면?

```java
Counter c1 = new Counter();

// Thread 1
c1.increment(); // count 변경 중

// Thread 2
Counter c2 = c1.clone(); // 동시에 복제
```
이 경우 `c2.count`가 1이 될 수도 있고, 여전히 0일 수도 있다.
즉, **clone()은 원본 상태를 정확히 복제하지 못할 수 있다.**

이런 불일치를 막기 위해서는
**clone() 호출 시점에 원본 객체의 상태가 변하지 않도록 보장해야 한다.**

그러나 현실적으로는 이런 문제까지 신경써야 해서 복잡해지므로,
멀티스레드 환경에서는 clone() 대신 복사 생성자나 불변 객체 설계가 훨씬 안전하다.

