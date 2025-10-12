# item 17. 변경 가능성을 최소화하라
> * 불변 객체는 설계의 기본 단위로 삼아라.
> * 가능한 한 클래스를 불변으로 만들고, 필요할 때만 가변 클래스로 만들어라.
>   * 꼭 변경해야할 필드 제외, 모두 `private final` 필드로 선언되어야한다
>   * 생성자와 정적 팩터리 제외하고 쵝화 메서드는 public 하면 안된다
> * 불변 객체는 공유가 안전하고, 캐싱과 재사용이 용이하다.

## 1. 불변 클래스의 정의

* **불변 클래스(Immutable Class)** 란 인스턴스의 내부 상태가 한 번 만들어지면 절대 변하지 않는 클래스이다.
* 생성된 후 객체의 상태를 변경할 수 없으며, 모든 연산은 **새로운 인스턴스**를 반환한다.
* 이렇게 객체를 변경하는 것이 아니라 새로운 객체를 반환하는 것을 **함수형 프로그래밍**이라고 한다.
```java
public final class Complex { 
    private final double re;
    private final double im;

    public Complex(double re, double im) { // final 클래스 대신 생성자를 priavet으로 두는 것도 방법
        this.re = re;
        this.im = im;
    }

    public double realPart() { return re; }
    public double imaginaryPart() { return im; }

    public Complex plus(Complex c) { // 함수형 메서드는 전치사를 사용하는 것이 관례
        return new Complex(re + c.re, im + c.im);
    }

    public Complex times(Complex c) {
        return new Complex(re * c.re - im * c.im,
                           re * c.im + im * c.re);
    }
}
```
---

## 2. 불변 클래스를 만드는 규칙

1. **객체의 상태를 변경하는 메서드를 제공하지 않는다.**

    * setter, 변경 메서드 등을 두지 않는다.
2. **모든 필드를 final로 선언한다.**

    * 한 번 초기화 후 다시 대입되지 않게 한다.
3. **클래스를 final로 선언하거나, 생성자를 private으로 두고 하위 클래스 생성을 막는다.**

    * 상속으로 인해 내부 불변식이 깨질 위험 방지.
4. **모든 필드를 private으로 선언한다.**

    * 외부에서 직접 접근하여 수정하는 것을 막는다.
5. **가변 객체를 참조하는 필드는 방어적 복사를 수행한다.**

    * 생성자, 접근자, 반환 시 모두 복사본을 사용해야 불변성 보장.

---

## 3. 불변 클래스의 장점

1. **안전하다 (Thread-safe)**

    * 상태가 변하지 않으므로 동기화 없이 여러 스레드에서 동시에 접근 가능.
2. **단순하고 예측 가능하다**

    * 값이 절대 변하지 않으므로 디버깅과 테스트가 쉽다.
3. **보안성이 높다**

    * 내부 상태를 외부에 노출해도 변하지 않으므로 조작 불가.
4. **해시코드 캐싱 가능**

    * hashCode는 한 번 계산해 캐싱해두면 동일 인스턴스는 항상 같은 값.
5. **불변 객체를 자유롭게 공유 및 캐싱 가능**

    * 같은 값은 같은 객체로 재사용할 수 있음 (예: Integer, Boolean 등).

---

## 4. 불변 클래스의 단점

1. **객체가 자주 만들어진다**

    * 상태 변경 시마다 새로운 객체를 생성해야 하므로, 메모리 낭비로 이어질 수 있다.
2. **연산이 많은 경우 비용이 커질 수 있다**

    * 예: `BigInteger`나 `String`의 반복 연산은 매번 새로운 객체를 만들어낸다.

→ 이런 경우에는 내부적으로 **가변 동반 클래스(Mutable Companion Class)** 를 제공한다.
예: `StringBuilder`는 `String`의 가변 버전이다.

---

## 5. 불변 클래스의 정적 팩터리와 캐싱

* 불변 객체는 상태가 절대 변하지 않으므로, **같은 값을 재사용**해도 문제가 없다.
* 이를 위해 정적 팩터리 메서드를 사용해 **캐싱된 인스턴스를 반환**할 수 있다.

```java
public final class Money {
    private static final Map<Integer, Money> CACHE = new HashMap<>();
    private final int amount;

    private Money(int amount) { this.amount = amount; }

    public static Money valueOf(int amount) {
        return CACHE.computeIfAbsent(amount, Money::new);
    }

    public int getAmount() { return amount; }
}
```

* `valueOf()`가 이미 존재하는 인스턴스를 반환하므로, 중복 생성 방지.
* 대표적인 예: `Boolean.valueOf()`, `Integer.valueOf()`, `BigInteger.ZERO`, `LocalDate.of()` 등.

---

## 6. 불변 클래스 설계 시 주의점

1. **가변 객체를 참조하지 않도록 주의**

   ```java
   public final class Period {
       private final Date start;
       private final Date end;

       public Period(Date start, Date end) {
           this.start = new Date(start.getTime()); // 방어적 복사
           this.end = new Date(end.getTime());
       }

       public Date start() {
           return new Date(start.getTime()); // 반환 시에도 복사
       }

       public Date end() {
           return new Date(end.getTime());
       }
   }
   ```
2. **상속보다는 컴포지션**

    * 하위 클래스에서 내부 불변식을 깨뜨릴 수 있으므로 상속은 금지.
3. **성능 문제가 심각하다면 가변 클래스 병행 설계**

    * `StringBuilder`, `BigInteger.Mutable` 같은 구조.

---

## 7. 함수형 프로그래밍과 불변 클래스
### 1) 절차형(Procedural) 프로그래밍 스타일

1. **특징**

    * 내부 상태를 변경하는 메서드 중심 설계
    * 명령(“이걸 이렇게 바꿔라”)을 순서대로 수행
    * 가변(mutable) 객체를 기반으로 작동

2. **예시**

   ```java
   class Counter {
       private int count = 0;

       public void increment() {
           count++;
       }

       public int getCount() {
           return count;
       }
   }
   ```

    * `increment()`가 상태를 직접 변경한다.
    * 외부 코드가 같은 인스턴스를 여러 스레드에서 동시에 조작하면 예측 불가능한 결과 발생 가능.
    * 동기화(synchronization)나 락(lock)으로 제어해야 안전함.

3. **문제점**

    * 동시성 문제에 취약
    * 테스트 및 디버깅 어려움 (상태 추적 필요)
    * 코드 재사용성 낮음 (상태 의존성 높음)

---

### 2. 함수형(Functional) 프로그래밍 스타일

1. **특징**

    * 상태 변경이 없음 (불변 객체 기반)
    * 메서드는 입력값을 받아 **새로운 객체**를 반환
    * “무엇을 할 것인가”에 집중 (선언형 접근)

2. **예시 (절차형 대비)**

   ```java
   public final class Counter {
       private final int count;

       public Counter(int count) {
           this.count = count;
       }

       public Counter increment() {
           return new Counter(count + 1);
       }

       public int getCount() {
           return count;
       }
   }
   ```

    * `increment()`는 기존 객체를 수정하지 않고, 새 객체를 반환한다.
    * 같은 입력에는 항상 같은 출력이 나오므로 **참조 투명성(referential transparency)** 유지.
    * 멀티스레드 환경에서도 동기화가 필요 없다.

3. **장점**

    * **스레드 안전(Thread-safe)**
      : 여러 스레드가 동시에 사용해도 상태 충돌이 없음.
      * **테스트 용이**
      : 외부 상태에 의존하지 않으므로, 순수 함수처럼 동작.
      * **코드 재사용 및 조합 용이**
      : 함수(메서드)끼리 조합 가능, 불변 객체 간 연산 안정적.

4. **단점**

    * 연산마다 새 객체를 만들기 때문에, **메모리와 GC 비용** 증가 가능.
    * 내부적으로 많은 객체가 생성되어, 성능 민감한 구간에서는 주의 필요.

---

### 3. 불변 클래스와 함수형 스타일의 관계

1. **불변 클래스 = 함수형 스타일의 기반**

    * 불변 객체는 내부 상태가 변하지 않기 때문에
      “입력 → 새로운 결과 반환” 형태의 함수를 구현하기에 적합하다.

2. **절차형에서 함수형으로의 전환 예시**

   ```java
   // 절차형 스타일
   BigInteger sum = BigInteger.ZERO;
   for (BigInteger val : list) {
       sum = sum.add(val);
   }
   ```

   ```java
   // 함수형 스타일 (Stream API 활용)
   BigInteger sum = list.stream()
                        .reduce(BigInteger.ZERO, BigInteger::add);
   ```

    * 절차형은 변수 `sum`을 계속 변경한다.
    * 함수형은 불변 객체를 사용해 연산 결과를 새로 생성한다.

3. **결과**

    * 함수형 스타일은 상태 의존도가 낮아 병렬 처리, 캐싱, 테스트에 유리.
    * 절차형 스타일은 성능적으로 단순하지만, 안정성과 유지보수성이 떨어짐.
