# item 29. 이왕이면 제네릭 타입으로 만들라
> - 타입 안전한 API를 제공하려면 이왕이면 제네릭 타입으로 만들어라.
> - 경고 억제가 불가피할 때는 최소 범위에 국한시켜라.

## 1. 핵심 개념

* **제네릭 타입(Generic type)** 은 클래스나 인터페이스의 **타입 매개변수(parameterized type)** 를 선언해서,
  클라이언트 코드에서 명시적 형변환 없이 **타입 안정성(type safety)** 을 확보하는 방법이다.

* “Object 기반” 코드 대신 “타입 매개변수 기반” 코드로 바꾸면 컴파일 타임에 오류를 잡을 수 있다.

---
## 2. Object 기반 Stack (변환 전)

```java
public class Stack {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(Object e) {
        elements[size++] = e;
    }

    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        Object result = elements[--size];
        elements[size] = null; // 메모리 누수 방지
        return result;
    }
}
```

문제점:

* `pop()` 호출 후 항상 `(String)` 등으로 형변환해야 함
* 잘못된 타입이 들어가도 **런타임 시점에만 오류 발생**

---

## 3. 제네릭 타입으로 변경

```java
public class Stack<E> {
    private E[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        elements = new E[DEFAULT_INITIAL_CAPACITY]; // 컴파일 오류
    }
}
```

→ `new E[]`는 불가능하다.
제네릭의 **타입 소거(type erasure)** 때문에, 런타임에는 `E`의 실제 타입 정보를 알 수 없기 때문이다.

---

## 4. 배열 처리의 두 가지 방법
### 방법 ①. Object 배열을 생성한 후 형변환하는 방법

```java
public class Stack<E> {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(E e) {
        elements[size++] = e;
    }

    @SuppressWarnings("unchecked")
    public E pop() {
        if (size == 0)
            throw new EmptyStackException();
        E result = (E) elements[--size];
        elements[size] = null;
        return result;
    }
}
```

* 내부 저장소는 `Object[]`
* 원소를 꺼낼 때 `(E)`로 형변환
* **타입 안정성이 보장됨 (push에서 이미 E 타입만 허용하기 때문)**
* 경고 억제가 불가피하므로 `@SuppressWarnings("unchecked")`는 **pop() 메서드 내부**에만 붙인다.

> 장점
>
> * 구현이 간단하고 안전하다.
> * 배열의 타입 안정성을 개발자가 명확히 제어할 수 있다.

> 단점
>
> * 내부 배열의 타입이 `Object[]`여서 외부에 노출되면 타입 안정성이 깨질 수 있다.
    >   (→ 반드시 `private`으로 유지해야 함)

---

### 방법 ②. 제네릭 배열을 생성하되, 형변환을 생성자에서 한 번만 수행하는 방법 << 더 선호 됨

```java
public class Stack<E> {
    private E[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    @SuppressWarnings("unchecked")
    public Stack() {
        elements = (E[]) new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(E e) {
        elements[size++] = e;
    }

    public E pop() {
        if (size == 0)
            throw new EmptyStackException();
        E result = elements[--size];
        elements[size] = null;
        return result;
    }
}
```

* `(E[]) new Object[...]` 형변환을 **생성자 한 곳에서만 수행**
* 그 이후로는 안전하게 `E` 배열처럼 사용 가능
* 경고 억제는 **생성자에만 한정**

> 장점
>
> * `E[]`로 유지되어 API 전반이 일관된 제네릭 타입을 사용한다.
> * 형변환이 한 번만 일어난다.

> 단점
>
> * 배열 생성 부분에서 비검사 형변환이 불가피하다.
> * 내부 구조를 잘못 노출하면 `ClassCastException` 가능성 존재.

---

## 5. 핵심 정리

* 제네릭 타입을 만들 때는 내부 구현에서 불가피한 비검사 형변환이 있을 수 있다.
  → **경고 억제는 최소 범위로**
* 배열보다는 리스트를 사용하는 것도 좋은 대안이다. (가능하다면 `List<E>`로 대체)
* 제네릭으로 리팩터링 시 **형변환 제거 + 컴파일 타입 안정성 확보**가 목적이다.
