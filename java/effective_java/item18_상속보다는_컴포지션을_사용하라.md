# item 18. 상속보다는 컴포지션을 사용하라
> * 상속은 캡슐화를 깨뜨리며, 내부 구현에 의존하는 불안정한 확장 방법이다.
> * 기존 클래스를 확장하려면, 그 클래스가 상속을 위한 설계와 문서화를 충분히 갖추었는지 확인해야 한다.
> * 대부분의 경우, 상속 대신 **컴포지션(위임)** 을 사용하라.
> * 재사용 가능한 위임 헬퍼(`ForwardingSet` 같은)를 만들어 **컴포지션을 쉽게 적용**하라.

## 1. 상속(extends)의 문제점

1. **상속은 캡슐화를 깨뜨린다.**

    * 하위 클래스는 상위 클래스의 **구현 세부사항**에 의존한다.
    * 상위 클래스 내부 동작이 바뀌면, 하위 클래스가 **의도치 않게 깨질 수 있다.**
    * 즉, 상속은 “is-a 관계”임에도 **코드 수정 시 결합도가 너무 높다.**

2. **상위 클래스의 내부 동작을 정확히 알아야 안전하게 상속할 수 있다.**

    * 하지만 대부분의 API 문서는 내부 호출 순서나 보조 메서드 호출 여부를 명시하지 않는다.
    * 결과적으로, **상속을 안전하게 사용하기 어렵다.**

3. **상위 클래스 확장을 막는 이유**

    * `String`, `BigDecimal`, `LocalDate` 등 많은 클래스가 `final`인 이유는
      하위 클래스가 내부 불변식을 깨트릴 수 있기 때문이다.

---

## 2. 잘못된 상속 예시: 계측 기능을 덧씌운 Set

```java
public class InstrumentedHashSet<E> extends HashSet<E> {
    private int addCount = 0;

    @Override
    public boolean add(E e) {
        addCount++;
        return super.add(e);
    }

    @Override
    public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c);
    }

    public int getAddCount() {
        return addCount;
    }
}
```

* 이 클래스는 `add`가 호출될 때마다 카운트를 세는 **계측(instrumentation)** 기능을 추가했다.
* 하지만, `HashSet.addAll()`이 내부적으로 `add()`를 호출하므로
  `addCount`가 **두 번 증가**하는 오류가 발생한다.

→ **상위 클래스의 구현 세부사항(HashSet이 addAll에서 add를 재사용함)에 의존했기 때문.**

---

## 3. 해결책: 컴포지션(Composition)

**핵심 아이디어**

> 상속 대신, 상위 클래스의 인스턴스를 **필드로 두고(public이 아닌 private으로)**,
> 그 인스턴스에 **기능을 위임(delegate)** 하라.

```java
public class InstrumentedSet<E> extends ForwardingSet<E> {
    private int addCount = 0;

    public InstrumentedSet(Set<E> s) {
        super(s);
    }

    @Override
    public boolean add(E e) {
        addCount++;
        return super.add(e);
    }

    @Override
    public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c);
    }

    public int getAddCount() {
        return addCount;
    }
}
```

* 내부의 `Set<E>`에게 동작을 위임하므로,
  상위 클래스의 내부 구현에 의존하지 않는다.
* 내부에 `HashSet`, `TreeSet`, `LinkedHashSet` 등 어떤 Set이 오든
  기능은 그대로 유지된다.

---

## 4. ForwardingSet의 역할

```java
public class ForwardingSet<E> implements Set<E> {
    private final Set<E> s;

    public ForwardingSet(Set<E> s) { this.s = s; }

    public void clear()               { s.clear(); }
    public boolean add(E e)           { return s.add(e); }
    public boolean remove(Object o)   { return s.remove(o); }
    public boolean contains(Object o) { return s.contains(o); }
    public int size()                 { return s.size(); }
    public Iterator<E> iterator()     { return s.iterator(); }
    // 나머지 메서드들도 모두 위임
}
```

* 매번 위임 코드를 작성하는 수고를 덜기 위해 만든 **재사용 가능한 헬퍼 클래스**.
* `InstrumentedSet`은 `ForwardingSet`을 상속만 하면 필요한 부분만 오버라이드 가능.
* 즉, **컴포지션을 쉽게 재사용할 수 있게 해주는 상속 구조**다.
* 구글 **Guava**의 `ForwardingList`, `ForwardingMap`, `ForwardingSet` 등이 같은 패턴이다.

---

## 5. 컴포지션의 장점

1. **캡슐화 유지**

    * 내부 객체의 동작 변경이 외부에 영향을 주지 않는다.

2. **유연성**

    * 런타임에 다른 인스턴스를 주입하여 동작을 바꿀 수 있다.

3. **안정성**

    * 상위 클래스의 수정이나 버전 차이에 영향을 받지 않는다.

4. **조합 가능성**

    * 여러 기능(로깅, 계측, 검증 등)을 **래핑 형태로 체이닝**할 수 있다.

---

## 6. 언제 상속을 써도 괜찮은가

1. 클래스가 **확장을 고려해 설계되고 문서화되어 있을 때**
2. **상속받은 클래스가 진정한 하위 타입(subtype)** 일 때
   → 예: `LinkedHashSet`은 `HashSet`의 정렬 버전으로 “is-a” 관계가 성립.
   - "B가 A와 `is-a` 관계라면" B가 A인가?에 YES 여야한다.
3. 상위 클래스의 **자기 자신의 메서드 호출(자기 호출)** 에 의존하지 않을 때.

그 외에는 **상속 대신 컴포지션**이 기본 선택이다.
