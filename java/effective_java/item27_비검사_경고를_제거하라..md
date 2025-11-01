# item 27. 비검사 경고를 제거하라

> * 가능한 한 **모든 비검사 경고를 제거하라.**
> * 제거할 수 없으면 `@SuppressWarnings("unchecked")`를 **안전하다고 확신하는 최소한의 범위**에만 사용하고 **이유를 주석으로 남겨라.**
> * 경고 없는 코드는 **타입 안정성이 입증된 코드**다.

## 1. 비검사 경고란

제네릭을 사용하면서 컴파일러가 타입 안전성을 완전히 검증하지 못할 때 발생하는 경고다.

예:

```java
List list = new ArrayList();       // raw type 사용
List<String> strings = list;       // unchecked assignment 경고 발생
```

* **"unchecked"**는 컴파일러가 타입을 *검증하지 못했다(unverified)* 는 뜻이다.
* 대부분의 경우 **타입 안정성을 해칠 수 있는 신호**이므로 무시하지 말아야 한다.

---

## 2. 경고를 제거해야 하는 이유

* 경고를 제거하면 **컴파일러가 타입 안전을 완전히 검증**할 수 있다.
* 경고가 남아 있으면 **실행 시 ClassCastException 위험**이 존재한다.
* 경고가 없는 코드는 **유지보수성과 신뢰성**이 높다.

---

## 3. 경고를 제거하는 방법

### (1) 제네릭 타입 명시

```java
List<String> list = new ArrayList<>(); // 타입 파라미터 명시
```

### (2) 제네릭 메서드로 감싸기

```java
public static <T> List<T> newList() {
    return new ArrayList<>();
}
```

### (3) @SuppressWarnings("unchecked") 최소 범위로 사용

* 정말 타입 안전하다고 확신할 때만 사용
* 가능한 한 **좁은 범위**에 적용 (변수나 문장 수준)
* 주석으로 "왜 안전한지" 반드시 설명해야 함

**잘못된 예시**
```java
@SuppressWarnings("unchecked")
public <T> T[] toArray(T[] a) {
    if (a.length < size)
        return (T[]) Arrays.copyOf(elements, size, a.getClass());
    System.arraycopy(elements, 0, a, 0, size);
    return a;
}
```
**좋은 예시**
```java
public <T> T[] toArray(T[] a) {
    if (a.length < size) {
        // 안전 근거: Arrays.copyOf가 a.getClass()를 사용해 런타임 타입을 보존한다.
        @SuppressWarnings("unchecked")
        T[] r = (T[]) Arrays.copyOf(elements, size, a.getClass());
        return r;
    }
    System.arraycopy(elements, 0, a, 0, size);
    if (a.length > size) a[size] = null;
    return a;
}
```
> `@SuppressWarnings("unchecked")`를 클래스 전체나 메서드 전체에 붙이는 것은 위험하다.
> 실제 문제 경고를 숨겨서 나중에 오류로 이어질 수 있다.

---
