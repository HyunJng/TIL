# item 20. 추상 클래스보다는 인터페이스를 우선하라
> - **타입 정의는 인터페이스를 우선하라.**
>    * 유연하고, 다중 구현 가능하며, 기존 코드에 손쉽게 추가할 수 있다.
> - **공통 구현이 필요하면 추상 골격 클래스를 함께 제공하라.**
>    * 인터페이스의 규약을 따르면서도 중복 구현을 줄일 수 있다.
> - **Forwarding 클래스는 컴포지션 기반 확장에 유용하다.**
>    * 상속 대신 위임으로 기능을 추가할 때 사용한다.
> - **기본 구현은 항상 보수적으로 설계하라.**
>    * `AbstractMapEntry.setValue`처럼 기본값을 “불변”으로 두는 것이 안전하다.


## 1. 인터페이스의 장점

1. **다중 구현 가능**

    * 클래스는 하나의 추상 클래스만 상속할 수 있지만, 여러 인터페이스를 동시에 구현할 수 있다.
    * 예: `class SingerSongwriter implements Singer, Songwriter`

2. **믹스인(Mixin) 역할 가능**

    * 기존 클래스에 특정 능력을 “혼합”할 수 있다.
      예: `Comparable`, `Serializable`, `Cloneable` 등은 모두 믹스인 인터페이스.

3. **기존 클래스에도 쉽게 추가 가능**

    * 이미 상속 계층에 속해 있어도 `implements`로 인터페이스를 붙일 수 있다.
    * 반면 추상 클래스는 상속 계층 제약으로 인해 추가 불가.

4. **계층 구조와 무관한 타입 정의**

    * `Singer`와 `Songwriter`처럼 독립적인 개념을 자유롭게 조합 가능.

---

## 2. 디폴트 메서드의 활용과 주의점

* Java 8부터 인터페이스도 `default` 메서드로 구현 코드를 포함할 수 있다.
* 하지만 다음과 같은 제약이 있다.

    * 상태(필드)를 가질 수 없다.
    * 다중 상속 시 이름 충돌이 발생할 수 있다.
* 따라서 **단순 보조 기능** 정도에만 사용하고, 복잡한 기본 로직은 추상 골격 클래스로 옮기는 것이 좋다.

---

## 3. 추상 클래스의 역할

* 공통된 **상태(필드)** 와 **기본 동작(메서드)** 을 함께 제공할 때 필요하다.
* 예: `AbstractList`, `AbstractMap`
* 단, 단일 상속 제약 때문에 신중히 사용해야 한다.

---

## 4. 인터페이스 + 추상 클래스 조합: 추상 골격 클래스 패턴

### 1) 개념
- 인터페이스로 타입 규약을 정의하고, 
- 추상 골격 클래스(Skeletal Implementation)로 기본 구현을 제공한다.

### 2) 구조

```
Interface → Abstract Skeletal Class → Concrete Implementation
```

### 3) 장점

| 항목     | 설명                                    |
| ------ | ------------------------------------- |
| 코드 재사용 | 인터페이스 구현 시 발생하는 중복 제거                 |
| 일관성    | 모든 구현체가 동일한 규약을 따름                    |
| 유연성    | 인터페이스의 다중 구현 가능성 + 추상 클래스의 코드 재사용성 결합 |

### 4) 클래스 이름 관례
골격클래스의 이름은 `Abstract + 인터페이스이름` 으로 짓는 것이 관례이다.

예시
* `Collection` + `AbstractCollection`
* `List` + `AbstractList`
* `Map` + `AbstractMap`

---

## 5. 추상 골격 클래스 작성 절차

1. **인터페이스 정의**

    * 타입 규약만 명세 (메서드 시그니처, 계약 조건)

2. **디폴트 메서드 추가 (선택)**

    * Java 8 이상에서는 간단한 기본 구현을 `default`로 제공 가능

3. **추상 골격 클래스 작성**

    * 복잡한 공통 동작(예: `equals`, `hashCode`, `toString`)을 구현
    * 필수 메서드만 추상 메서드로 남긴다.

4. **구체 클래스 구현**

    * 핵심 동작만 구현 (`getKey`, `getValue` 등)

### 1) 예시: `Map.Entry`

```java
public interface MapEntry<K, V> {
    K getKey();
    V getValue();
    V setValue(V value);
}
```

```java
public abstract class AbstractMapEntry<K, V> implements MapEntry<K, V> {
    @Override
    public V setValue(V value) {
        throw new UnsupportedOperationException();
    }

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof MapEntry))
            return false;
        MapEntry<?, ?> e = (MapEntry<?, ?>) o;
        return Objects.equals(getKey(), e.getKey())
            && Objects.equals(getValue(), e.getValue());
    }

    @Override
    public int hashCode() {
        return Objects.hashCode(getKey()) ^ Objects.hashCode(getValue());
    }
}
```

### 2) 왜 `setValue`는 추상이 아니라 예외를 던질까?

* 대부분의 `Map.Entry` 구현은 **불변(immutable)** 이다.
  따라서 기본 구현이 “수정 불가능”하도록 설정하는 것이 더 안전하다.
* 필요할 때만 명시적으로 `setValue`를 재정의하여 수정 기능을 허용할 수 있다.
* 추상으로 남기면 모든 하위 클래스가 불필요하게 `throw` 코드를 반복해야 하므로,
  **기본 구현을 “보수적(안전)”으로 제공하는 것이 합리적**이다.

---

## 6. Forwarding 클래스와 골격 클래스의 차이

| 구분    | 추상 골격 클래스 (`Abstract*`)      | Forwarding 클래스 (`Forwarding*`)    |
| ----- | ---------------------------- | --------------------------------- |
| 목적    | 인터페이스 기본 구현 제공 (상속 기반)       | 위임(forwarding) 코드 재사용 (컴포지션 기반)   |
| 대표 패턴 | 템플릿 메서드 패턴                   | 데코레이터 패턴                          |
| 상속 여부 | 추상 클래스                       | 구체 클래스                            |
| 사용 맥락 | 인터페이스 구현 시 코드 중복 방지          | 래퍼 클래스(컴포지션)에서 위임 단순화             |
| 예시    | `AbstractSet`, `AbstractMap` | `ForwardingSet`, `ForwardingList` |

→ `ForwardingSet`은 “추상 골격 클래스처럼 보이지만”,
**상속이 아닌 컴포지션을 돕는 도우미 클래스**이다.

---

## 7. 래퍼 클래스 관용구와의 관계

* **래퍼(Wrapper) / 데코레이터(Decorator)** 패턴은 Item 18에서 다뤘다.
* 인터페이스와 함께 쓰면 매우 강력하다:

  ```java
  public class LoggingList<E> implements List<E> {
      private final List<E> list;
      public LoggingList(List<E> list) { this.list = list; }

      @Override
      public boolean add(E e) {
          System.out.println("added: " + e);
          return list.add(e);
      }
      // 나머지는 모두 list로 위임
  }
  ```
* 만약 타입이 추상 클래스로 정의돼 있었다면?
  → 이미 상속 계층이 있다면 확장이 불가능하다.
  → 인터페이스 기반 설계 덕분에 이런 제약에서 자유로울 수 있다.
