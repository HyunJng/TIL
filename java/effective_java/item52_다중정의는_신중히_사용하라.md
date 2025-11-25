# item 52. 다중정의(overloading)는 신중히 사용하라
> 1. 오버로딩은 **정적으로 결정되기 때문에** 예상과 다르게 작동할 수 있다.
> 2. API 사용자가 헷갈릴 여지가 있으면 **오버로딩을 피하는 것이 최선**이다.
> 3. 이름을 새로 정의하거나, 파라미터 타입을 명확히 제한하는 방식으로
   코드의 의미를 분명히 하는 것이 중요하다.
> 4. 오버라이딩과 달리 오버로딩은 다형성을 제공하지 않는다. 이를 항상 기억해야 한다.

## 1. 핵심 내용

* **오버로딩은 정적으로 선택되는 메서드**이다.
  즉, **컴파일 시점에** 어떤 메서드가 호출될지 결정된다.
* **오버라이딩은 동적 디스패치(dynamic dispatch)** 이다.
  즉, **런타임에 실제 객체 타입**에 따라 실행될 메서드가 결정된다.
* 이 차이 때문에, **오버로딩은 개발자가 예상하지 못한 결과를 유발**할 수 있다.
* 특히 인수 타입에 따라 메서드가 달라지는 API를 만들면 **혼동과 오류의 근원**이 되므로 피하는 것이 좋다.

---

## 2. 오버로딩이 위험한 이유

### 2.1. 컴파일 시점에 결정되기 때문

```java
void print(CharSequence s) { ... }
void print(String s) { ... }

CharSequence cs = "hello";
print(cs); // print(CharSequence) 호출
```

`cs`는 런타임에 String이지만, **정적 타입이 CharSequence이기 때문에** String 버전이 호출되지 않는다.

### 2.2. 다형성처럼 보이지만 다형성이 아님

* 오버로드된 메서드는 타입 계층과 무관하게 **“적절한 시그니처를 컴파일러가 골라줌”**이 전부.
* 따라서 개발자가 의도하는 다형성이 구현되지 않음.

### 2.3. 컬렉션 예시

책에서 유명한 예시:

```java
public class CollectionClassifier {
    public static String classify(Set<?> s)       { return "Set"; }
    public static String classify(List<?> lst)    { return "List"; }
    public static String classify(Collection<?> c){ return "Unknown"; }
}

Collection<?>[] collections = {
    new HashSet<String>(),
    new ArrayList<Integer>(),
    new HashMap<String, String>().values()
};

for (Collection<?> c : collections)
    System.out.println(classify(c));
```

출력:

```
Unknown
Unknown
Unknown
```

이유:

* `c`의 정적 타입은 항상 `Collection<?>`
* 따라서 오버로드된 `classify(Set)`나 `classify(List)`는 고려되지 않음

---

## 3. 오버라이딩과의 비교

### 3.1. 오버라이딩 = 동적 디스패치 (런타임 결정)

상속 관계라면 다음이 기대대로 동작한다.

```java
class A { void hello() { ... } }
class B extends A { void hello() { ... } }

A a = new B();
a.hello(); // B의 hello 실행 (다형성)
```

### 3.2. 오버로딩 = 정적 디스패치 (컴파일 타임 결정)

```java
void hello(A a) { ... }
void hello(B b) { ... }

A a = new B();
hello(a); // A 버전 호출
```

`a`는 실제로 B이지만 정적 타입이 A라서 A 버전 호출됨.

---

## 4. 언제 오버로딩을 피해야 하는가

1. **매개변수가 서로 다른 타입 계층에 속하는 경우**
   → 혼란을 유발함
   예: `print(int)` vs `print(long)` vs `print(Integer)`
2. **null을 받을 수 있는 경우**
   → 어떤 메서드가 선택될지 직관적이지 않음
3. **API의 의미가 인수 타입에 따라 달라지는 경우**
   → 오버라이딩처럼 보이지만 실제로는 오버로딩이라 위험

---

## 5. null 인수 문제 (정적 디스패치 우선순위)

`null` 인수는 **가장 구체적인 타입**을 가진 메서드가 선택된다.

```java
void run(Object o)
void run(String s)

run(null); // run(String) 선택
```

컴파일러는 “String이 Object보다 더 구체적이라고 판단”하기 때문에 발생.

---

## 6. 안전하게 사용하는 방법

### 6.1. 서로 다른 기능을 하는 메서드에 오버로딩을 사용하지 말 것

```java
read(File f)
read(String path)
```

→ 같은 개념을 표현하지만 타입이 다르면 혼란 줄 수 있음
→ static factory method 패턴으로 해결 가능
예: `Path.of(String)`

### 6.2. 가변인수(varargs)와 함께 사용 시 특히 주의

가변인수 오버로딩은 컴파일러가 ambiguous error를 내거나 오해를 일으키는 경우가 많다.

### 6.3. 타입이 넓고 좁은 버전이 섞인 경우 가능하면 **이름을 완전히 다르게 하라**

```java
parseFromString(...)
parseFromBytes(...)
```

처럼 명시적으로 구분.

### 6.4 오버로딩을 하고 싶다면 똑같은 기능을 하도록 하라
* 어떤 메서드가 불리는지 몰라도 기능이 똑같다면 신경쓸게 없다.