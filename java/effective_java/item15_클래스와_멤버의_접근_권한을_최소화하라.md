# item 15. 클래스와 멤버의 접근 권한을 최소화하라
> 캡슐화의 핵심은 접근 제한자를 제대로 활용하는 것이다.
> 
> **“모든 요소는 가능한 한 최소한의 접근 권한을 가져야 한다.”**

## 1. 정보 은닉, 캡슐화의 장점
캡슐화를 한다는 것은 모든 내부 구현을 완벽히 숨겨 구현과 API를 분리한다는 뜻이다.
즉, **오직 API를 통해 다른 컴포넌트와 소통하며 서로의 내부동작에는 전혀 개의치 않는다**.
- 여러 컴포넌트를 병렬로 개발할 수 있어 개발 속도를 높인다.
- 디버깅 범위를 줄여 유지보수 비용을 낮춘다.
- 컴포넌트의 재사용성을 높인다.
- 컴포넌트 별로 테스트를 할 수 있기에 큰 시스템의 제작 난이도를 낮춘다.

**캡슐화를 제대로 활용하려면 접근제한자를 가능한 한 좁혀야한다**.

## 2. 접근 제한자의 기본 원칙
* **가장 낮은 접근 수준**을 부여하라.
* public으로 공개해야 하는 정당한 이유가 없다면 전부 private으로 선언하라.

| 제한자                           | 접근 가능 범위        | 사용 목적                      |
| ----------------------------- | --------------- | -------------------------- |
| `private`                     | 같은 클래스 내부만      | **가장 강력한 은닉**. 외부 접근 완전 차단 |
| `package-private`             | 같은 패키지          | 패키지 내부에서만 공유               |
| `protected`                   | 같은 패키지 + 하위 클래스 | **상속 관계에서만 제한적 공개**        |
| `public`                      | 모든 곳에서 접근 가능    | 외부 API로 공개                 |

---

## 3. 공개(public) API 최소화하기

* **내부 구현 클래스는 절대 public으로 두지 말라.**
  * 공개 API가 되는 순간 영원히 관리해주어야한다. 
  * 내부구현 클래스는 **package-private**으로 두어 내부에서만 사용하도록 한다.
* public 클래스의 **멤버(필드, 메서드, 중첩 클래스)** 도 가능한 한 접근을 제한한다.

> **설계 TIP**
> 1. 공개 API를 세심히 설계하고 그 외의 모든 멤버는 private으로 선언
> 2. 같은 패키지의 다른 클래스가 접근해야하는 멤버에 한하여 private을 package-priavte으로 풀어주기
> 3. 권한을 푸는 일이 자주 있다면 컴포넌트를 더 분해할 것을 고민해보기 
---

## 3. `protected`는 신중하게 사용하라

* `protected`는 내부 구현을 하위 클래스에 **노출하는 행위**이므로, 사실상 **API를 공개하는 것과 큰 차이가 없다**.
* 따라서 일반적으로는 **package-private이 더 낫다**.

---

## 4. public 필드는 절대 피하라

* **public 필드를 두면 불변성(invariant)을 보장할 수 없다.**
* 특히 `public non-final` 필드는 외부에서 자유롭게 수정 가능하므로 **객체 불변성이 깨짐**.
* `public static final` 상수만 예외적으로 허용된다.

  * 단, **불변 객체(immutable object)** 여야 한다.

```java
// 상수만 허용
public static final int MAX_SIZE = 100;

// 가변 객체는 안 됨
public static final List<String> list = new ArrayList<>(); // 외부에서 수정 가능
```

→ 가변 객체의 경우 `Collections.unmodifiableList()` 등으로 감싸야 함.

```java
private static final List<String> VALUES = List.of("A", "B");
public static final List<String> UNMODIFIABLE_VALUES = Collections.unmodifiableList(VALUES);
```

---

## 5. 클래스의 접근 수준

* 톱레벨 클래스(Top-level class)는 `public` 또는 package-private만 가능.
* **외부에서 사용할 필요가 없다면 package-private으로 두는 것이 정답**.

---

## 6. 중첩 클래스의 접근 제한

* 중첩 클래스(Inner class, Nested class)도 가능한 한 접근을 좁혀라.
* **private static class**로 두면 외부에 완전히 숨길 수 있다.
* **멤버 클래스보다 지역 클래스, 익명 클래스가 더 은닉 수준이 높다.**

---

## 7. 테스트와 접근제한의 균형

* 테스트를 위해 `private`을 `protected`로 바꾸는 건 바람직하지 않다.
  대신 **같은 패키지 안에 테스트를 두어 package-private 멤버를 접근**하도록 한다.