# item 21. 인터페이스는 구현하는 쪽을 생각해 설계하라
> **인터페이스는 ‘구현자 입장’에서도 안전하게 동작하도록 설계해야 한다.**
> 특히 이미 사용 중인 인터페이스를 진화시킬 때(default 메서드 추가 등),
> **기존 구현체의 불변식, 동기화 정책, 성능 특성**을 해치지 않도록 주의해야 한다.

## 1. 인터페이스는 '클라이언트' 뿐만 아니라 '구현자'의 계약이기도 하다

* 인터페이스를 **구현하는 사람(implementer)** 은 그 계약을 지켜야 하고,
* 인터페이스를 **호출하는 사람(caller)** 은 그 계약에 의존할 수 있어야 한다.

그래서 인터페이스 설계자는 두 방향 모두 고려해야 한다.

| 관점        | 고려 대상                                                             |
| --------- | ----------------------------------------------------------------- |
| **클라이언트** | 사용하기 쉬운가? 오용하기 어렵게 되어있는가?                                         |
| **구현자**   | 구현하기 쉬운가? 기존 클래스 구조를 과도하게 제한하지는 않는가? 새로운 요구사항이 생겨도 안정적으로 확장 가능한가? |

---

## 2. default 메서드는 구현자 입장에서 함정이 될 수 있다.

Java 8 이후, 기존 인터페이스에 **default 메서드**를 추가할 수 있게 되었다.
이건 겉보기엔 “구현자에게 편의”를 주는 기능이지만, 구현자 입장에서 함정이 될 수 있다.

### 1) Collection의 default 메서드인 `removeIf`

```java
interface Collection<E> {
    default boolean removeIf(Predicate<? super E> filter) {
        Objects.requireNonNull(filter);
        boolean removed = false;
        final Iterator<E> each = iterator();
        while (each.hasNext()) {
            if (filter.test(each.next())) {
                each.remove();
                removed = true;
            }
        }
        return removed;
    }
}
```
이 메서드가 추가되어
모든 `Collection` 구현체(예: `ArrayList`, `HashSet`, `TreeSet`, `SynchronizedCollection`)가
자동으로 이 메서드를 가지게 되었다.

하지만 문제는,
이 구현이 **기존 클래스의 내부 규약(invariant)이나 동기화 정책**과 맞지 않을 수도 있다는 것이다.

### 2) SynchronizedCollection의 removeIf 문제
* `Collections.synchronizedCollection()`은 내부적으로
  모든 메서드가 `synchronized (mutex)` 블록으로 감싸진 래퍼 클래스이다.
* 하지만 **default 메서드는 인터페이스에 직접 구현되어 있으므로**,
  해당 래퍼의 락 정책을 따르지 않는다.
* 따라서 `removeIf`가 자동 상속될 경우 다음과 같은 문제가 생긴다.

```java
Collection<Integer> syncList =
    Collections.synchronizedCollection(new ArrayList<>());

syncList.removeIf(x -> x < 0); // 락 없이 진행
```
→ 내부적으로 락을 잡지 않기 때문에 **ConcurrentModificationException**이나
데이터 불일치가 발생할 수 있다.

즉, “새 default 메서드 하나 추가”가 기존 수많은 구현체에 **예상치 못한 부작용**을 일으킬 수 있다.

### 3) removeIf의 해결
* JDK에서는 이후에 `SynchronizedCollection`이 `removeIf`를 **명시적으로 재정의**했다.

```java
public boolean removeIf(Predicate<? super E> filter) {
    synchronized (mutex) {
        return c.removeIf(filter);
    }
}
```

* 즉, default 메서드를 그대로 쓰면 위험했기 때문에
  기존 구현체가 재정의하여 동기화 정책을 유지한 것이다.

## 3. 설계 교훈

* **default 메서드는 만능이 아니다.**
* 인터페이스의 진화가 기존 구현체의 **불변식, 스레드 안정성, 성능 특성**을
  깨뜨릴 수 있음을 항상 고려해야 한다.
* 특히 `default` 메서드는 **기존 클래스의 내부 구조나 제약조건을 알 수 없기 때문에**,
  호환성 보장을 신중하게 검토해야 한다.
* 이미 널리 사용되는 인터페이스라면
  **새 메서드를 추가하지 않는 것이 가장 안전한 선택**이다.
