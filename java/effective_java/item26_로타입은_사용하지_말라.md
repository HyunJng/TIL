# item 26. 로 타입(raw type)은 사용하지 말라

> * 제네릭은 컴파일 시점 타입 검증을 제공 → **런타임 에러를 컴파일 타임에 방지**
> * 로 타입은 사용하지 말 것 → 제네릭의 타입 안전성 무력화
> * **대신 `<?>` 와일드카드 타입**을 사용하자
> * 단, `instanceof`나 `Class` 리터럴에서는 로타입 예외적으로 허용

## 1. 로 타입이란?

**로 타입(raw type)** 은 **제네릭 클래스나 인터페이스에서 타입 매개변수를 전혀 지정하지 않은 타입**을 말한다.

```java
List list = new ArrayList(); // 로 타입
List<Object> list = new ArrayList<>(); // 제네릭 타입
```

* `List`는 **로 타입**
* `List<Object>`는 **모든 타입의 원소를 담을 수 있는 제네릭 타입**

즉, 제네릭 이전(Java 5 이전) 코드와의 호환성을 위해 **컴파일러가 허용**하지만,
**타입 안정성(type safety)** 이 보장되지 않는다.

---

## 2. 로 타입의 문제점

로 타입을 사용하면 **컴파일러의 타입 검사 기능이 무력화된다.**

```java
List list = new ArrayList();
list.add("string");
list.add(123);  // 다른 타입도 추가 가능 (컴파일러가 경고하지 않음)
```

이후 꺼낼 때 문제가 발생한다:

```java
for (Object o : list) {
    String s = (String) o; // ClassCastException 발생 가능
}
```

→ 제네릭의 도입 목적(타입 안정성 확보, 캐스팅 제거)이 완전히 사라진다.

---

## 3. 제네릭 타입을 써야 하는 이유

제네릭을 사용하면 **컴파일 시점**에 타입 오류를 잡을 수 있다.

```java
List<String> list = new ArrayList<>();
list.add("string");
list.add(123); // 컴파일 에러!
```

→ 런타임이 아닌 **컴파일 단계에서 타입 불일치 방지**

---

## 4. 로 타입 vs 비한정 와일드카드 타입

로 타입을 써야 할 것 같을 때는 **대신 와일드카드 타입을 사용하자.**

```java
void printList(List<?> list) {
    for (Object o : list) {
        System.out.println(o);
    }
}
```

* `List<?>`는 “어떤 타입의 리스트든 받는다”는 뜻
* 단, **읽기 전용**으로만 사용 가능 (`add` 불가)

===
## 5. `List<Object>` vs `List<?>`의 차이

| 구분         | `List<Object>`             | `List<?>`                 |
| ---------- | -------------------------- | ------------------------- |
| 의미         | **Object 타입만 담을 수 있는 리스트** | **어떤 타입의 리스트든 참조 가능**     |
| 추가 (`add`) | `add(Object)` 가능           | `add()` 불가능 (컴파일 에러)      |
| 읽기 (`get`) | `Object` 반환                | `Object` 반환               |
| 대입 가능 여부   | `List<String>`은 대입 불가      | `List<String>` 대입 가능      |
| 요약         | “**모든 타입의 원소를 담는 리스트**”    | “**타입을 모르는 리스트** (읽기 전용)” |

### 예시

```java
List<Object> objects = new ArrayList<>();
objects.add("string");  // 가능
objects.add(1);         // 가능

List<String> strings = List.of("a", "b");
// objects = strings;   // 불가능 (타입 불일치)

List<?> anyList = strings;
Object o = anyList.get(0); // 읽기 가능
anyList.add("c");          // 컴파일 에러 (추가 불가)
```

즉,

* `List<Object>`는 **모든 타입의 객체를 넣을 수 있는 “수용용기”**,
* `List<?>`는 **타입을 알 수 없는 “읽기 전용 뷰(View)”**라고 이해하면 된다.

---

## 6. 로 타입이 필요한 경우 (예외)

제네릭 등장 이전 코드와의 **호환성(backward compatibility)** 때문에 **일부 예외적 상황에서만 허용**된다.

1. **`instanceof` 검사**

   ```java
   if (obj instanceof Set) { // OK
       Set<?> s = (Set<?>) obj; // 안전한 캐스팅
   }
   ```

2. **클래스 리터럴(Class Literal)**

   ```java
   Class<List> listClass = List.class; // OK
   Class<List<String>> stringListClass = List<String>.class; // 불가능
   ```
