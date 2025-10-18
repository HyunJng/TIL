# item 24. 멤버 클래스는 되도록 static으로 만들라
> - **멤버 클래스는 기본적으로 static으로 만들라.**
>     → 외부 인스턴스에 접근할 필요가 없다면 반드시 static.
> - **비정적 멤버 클래스는 외부 인스턴스와 강하게 연결되어야만 한다.**
>   → 주로 외부 객체의 내부 상태를 “다른 형태로 보여주는 뷰(adapter)”일 때만 적합.
> - **불필요한 외부 참조는 메모리 누수를 유발할 수 있다.**
>   → 내부 클래스가 외부 객체보다 오래 살아남는 경우 주의.

## 1. 멤버 클래스의 종류

중첩 클래스(nested class)는 다른 클래스 내부에 정의된 클래스를 말한다.
이는 크게 **4가지**로 나뉜다.

* **정적 멤버 클래스 (static member class)**
* **비정적 멤버 클래스 (non-static member class, inner class)**
* **익명 클래스 (anonymous class)**
* **지역 클래스 (local class)**

이 중 **멤버 클래스(member class)**란,
이름이 있고 다른 클래스 내부에 정의된 클래스로, 정적(static)과 비정적(non-static) 두 종류가 있다.

---

## 2. 정적 멤버 클래스 (static member class)

* 외부 클래스와 **논리적으로만 관련**되어 있으며, **외부 인스턴스와 직접적인 연결은 없다.**
* 따라서 외부 클래스의 인스턴스 없이도 생성 가능하다.
* 바깥 클래스의 인스턴스 필드에 접근할 수 없으며,
  일반적인 독립 클래스처럼 동작하지만, 외부 클래스의 이름공간 안에 들어가 있어 **캡슐화를 강화**할 수 있다.

```java
class Outer {
    static class StaticMember {
        void print() { System.out.println("정적 멤버 클래스"); }
    }
}
```

**사용 예시**

* `Enum` 내부의 헬퍼 클래스
* 유틸리티 성격의 클래스
* 외부 클래스와 **독립적으로 동작 가능한 클래스**

> 외부 인스턴스와 독립적이라면 항상 static으로 선언하라.

---

## 3. 비정적 멤버 클래스 (non-static member class)

* 외부 클래스의 인스턴스와 **암묵적으로 연결(inner)** 된다.
* 인스턴스 생성 시 반드시 외부 클래스의 인스턴스가 존재해야 한다.

  ```java
  Outer outer = new Outer();
  Outer.Inner inner = outer.new Inner();
  ```
* 내부적으로 `Outer.this` 참조를 암묵적으로 가지고 있어
  외부 클래스의 필드나 메서드에 접근할 수 있다.

**주의점**

* 외부 인스턴스를 참조하므로 **GC가 외부 객체를 수거하지 못해 메모리 누수가 발생할 수 있다.**
* 따라서 **필요한 경우가 아니면 static으로 만들어야 한다.**

---

## 4. 비정적 멤버 클래스의 적절한 사용 예: 어댑터(Adapter)

비정적 멤버 클래스는 오직 **외부 인스턴스에 종속적인 뷰(view)** 를 제공해야 할 때만 사용해야 한다.

> 외부 클래스의 내부 상태를 다른 인터페이스 형태로 변환해 노출할 때 — 즉, **어댑터** 역할일 때 적합하다.

### 예시: `HashMap.EntrySet`

```java
public class HashMap<K, V> {
    private final class EntrySet extends AbstractSet<Map.Entry<K,V>> {
        public Iterator<Map.Entry<K,V>> iterator() {
            return new EntryIterator();
        }
        public int size() { return size; }
    }

    public Set<Map.Entry<K, V>> entrySet() {
        return new EntrySet();
    }
}
```

* `EntrySet`은 `HashMap`의 내부 자료구조를
  `Set<Map.Entry<K,V>>` 인터페이스로 감싸주는 **뷰(view)** 이다.
* `EntrySet`에서의 모든 조작(`remove`, `clear` 등)은
  원본 `HashMap`에도 동일하게 반영된다.
* 이런 양방향 연결을 위해서는 외부 인스턴스(`HashMap`)의 필드에 접근할 수 있어야 하므로
  **비정적 멤버 클래스**로 구현된다.

---

## 5. 익명 클래스 / 지역 클래스

* **익명 클래스**: 이름이 없으며, 즉석에서 객체 생성과 함께 정의되는 클래스.
  주로 간단한 콜백, Comparator, Runnable 등을 정의할 때 사용된다.
* **지역 클래스**: 메서드 내부에 정의된 이름 있는 클래스.
  사용 범위가 좁고 간단한 경우에만 사용된다.

> 최근에는 대부분 람다(lambda)로 대체되는 경우가 많다.