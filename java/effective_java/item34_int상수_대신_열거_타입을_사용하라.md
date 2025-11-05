# item 34. int 상수 대신 열거 타입을 사용하라
> * `enum`은 클래스이므로, 각 상수가 **자신의 이름 공간과 타입**을 가진다.
> * `enum`을 사용하면 **타입 안전성**, **가독성**, **유지보수성**이 모두 향상된다.
> * **상수 집합을 표현할 때는 항상 열거 타입을 우선 고려하라.**
## 1. 열거 패턴(Enum pattern)의 문제점

과거에는 상수를 표현하기 위해 **정수 상수(static final int)** 를 사용했다.

```java
public class Apple {
    public static final int FUJI = 0;
    public static final int PIPPIN = 1;
}

public class Orange {
    public static final int NAVEL = 0;
    public static final int TEMPLE = 1;
}
```

이런 식으로 상수를 정의하는 방식을 **열거 패턴(enumeration pattern)** 이라고 한다.
하지만 이 방식에는 심각한 단점이 있다.

1. **타입 안전성이 없다.**
   서로 다른 상수 집합(예: Apple과 Orange)을 혼용해도 컴파일러가 잡아주지 않는다.

   ```java
   int apple = Apple.FUJI;
   int orange = Orange.NAVEL;
   apple = orange;  // 오류 아님!
   ```

2. **이름 공간이 오염된다.**
   모든 상수가 한 전역 공간에 존재하므로, 이름 충돌을 피하기 위해 접두사(`APPLE_`, `ORANGE_`)를 강제로 붙여야 한다.

3. **디버깅이 어렵다.**
   상수의 실제 값은 단순한 숫자이므로, 출력 시 의미를 알 수 없다.

   ```java
   System.out.println(Apple.FUJI); // "0" 출력
   ```

4. **상수 값이 바뀌면 바이너리 호환성(binary compatibility)이 깨진다.**
   기존에 컴파일된 다른 코드가 여전히 예전 숫자에 의존하고 있을 수 있다.

5. **확장성이나 기능 추가가 어렵다.**
   각 상수는 단순한 숫자일 뿐이므로, 상태나 메서드를 부여할 수 없다.

---

## 2. 열거 타입(enum)의 등장

이러한 문제를 해결하기 위해 Java 5부터 **열거 타입(enum type)** 이 도입되었다.
열거 타입은 **고정된 상수 집합을 나타내는 타입**으로, 다음과 같이 선언한다.

```java
public enum Apple { FUJI, PIPPIN, GRANNY_SMITH }
public enum Orange { NAVEL, TEMPLE, BLOOD }
```

### 특징

* `enum`은 **컴파일 시점에 클래스**로 변환된다.

  ```java
  public final class Apple extends Enum<Apple> {
      public static final Apple FUJI = new Apple("FUJI", 0);
      public static final Apple PIPPIN = new Apple("PIPPIN", 1);
      public static final Apple GRANNY_SMITH = new Apple("GRANNY_SMITH", 2);
  }
  ```

* 즉, `enum`은 **각자의 이름 공간(namespace)** 을 가지며,
  이름이 같은 상수라도 다른 열거 타입이면 **서로 충돌하지 않는다.**

  ```java
  Apple.FUJI != Orange.FUJI // 완전히 다른 타입
  ```

---

## 3. 열거 타입의 장점

1. **타입 안전성**
   각 상수는 자신이 속한 열거 타입의 인스턴스로만 존재하므로,
   `Apple` 타입의 값에 `Orange`를 넣을 수 없다.

2. **이름 공간 분리**
   각 열거 타입이 **자신만의 이름 공간을 가지므로**,
   이름이 같은 상수도 서로 다른 열거 타입 안에서 공존할 수 있다.

3. **상수 값이 내부적으로 관리되어 외부 노출되지 않는다.**
   정수 상수처럼 특정 값(0,1,2)에 의존하지 않으므로,
   값 변경에 따른 호환성 문제도 발생하지 않는다.

4. **문자열로 표현 가능(toString 지원)**
   디버깅 시 `FUJI` 같은 이름이 그대로 출력된다.

5. **switch문과 함께 사용 가능**

   ```java
   switch (apple) {
       case FUJI -> ...
       case PIPPIN -> ...
   }
   ```

---

## 4. 열거 타입에 데이터와 메서드 추가하기

열거 타입은 단순한 상수 집합 그 이상이다.
각 상수에 **고유한 데이터와 동작을 부여**할 수도 있다.

```java
public enum Planet {
    MERCURY(3.302e+23, 2.439e6),
    VENUS(4.869e+24, 6.052e6),
    EARTH(5.975e+24, 6.378e6);

    private final double mass;   // 질량 (kg)
    private final double radius; // 반지름 (m)
    private final double surfaceGravity; // 표면 중력 (m/s^2)

    private static final double G = 6.67300E-11;

    Planet(double mass, double radius) {
        this.mass = mass;
        this.radius = radius;
        this.surfaceGravity = G * mass / (radius * radius);
    }

    public double surfaceWeight(double mass) {
        return mass * surfaceGravity;
    }
}
```

이제 각 행성 상수가 **고유한 상태와 메서드**를 가지며,
완전한 객체로 동작한다.

```java
double earthWeight = 70;
double mass = earthWeight / Planet.EARTH.surfaceGravity;
for (Planet p : Planet.values())
    System.out.printf("Weight on %s is %f%n", p, p.surfaceWeight(mass));
```

---

## 5. 상수별 동작을 다르게 하기

각 상수마다 동작이 다를 경우,
**상수별 메서드 구현(constant-specific method implementation)** 을 사용할 수 있다.

```java
public class Main {
    public enum Operation{
        PLUS((x, y) -> x + y),
        MINUS((x, y) -> x - y);

        private BiFunction<Integer, Integer, Integer> operation;

        Operation(BiFunction<Integer, Integer, Integer> operation) {
            this.operation = operation;
        }

        public int apply(int x, int y) {
            return this.operation.apply(x, y);
        }
    }
    
    public static void main(String[] args) throws Exception {
        System.out.println("plus = " + Operation.PLUS.apply(2, 3));;
    }
}
```

`switch` 문보다 객체지향적으로 확장 가능하며,
새 연산을 추가할 때 기존 코드를 수정하지 않아도 된다.

---

## 6. 전략 열거 타입(enum strategy pattern)

공통된 열거 타입 내부에서,
서로 다른 **전략(strategy)** 을 적용할 수도 있다.

예: 급여 계산 정책

```java
public enum PayrollDay {
    MONDAY(PayType.WEEKDAY),
    SATURDAY(PayType.WEEKEND);

    private final PayType payType;
    PayrollDay(PayType payType) { this.payType = payType; }

    double pay(double hoursWorked, double payRate) {
        return payType.pay(hoursWorked, payRate);
    }

    private enum PayType {
        WEEKDAY {
            double overtimePay(double hrs, double payRate) {
                return hrs <= 8 ? 0 : (hrs - 8) * payRate / 2;
            }
        },
        WEEKEND {
            double overtimePay(double hrs, double payRate) {
                return hrs * payRate / 2;
            }
        };
        abstract double overtimePay(double hrs, double payRate);

        double pay(double hoursWorked, double payRate) {
            double basePay = hoursWorked * payRate;
            return basePay + overtimePay(hoursWorked, payRate);
        }
    }
}
```

이 방식은 **상수별 메서드 구현**을 사용할 때
`switch` 문으로 모든 경우를 나열해야 하는 문제를 해결한다.

---

## 7. 열거 타입의 추가 기능

* **`values()`**: 모든 상수를 배열로 반환
* **`valueOf(String name)`**: 문자열을 해당 상수로 변환
* **`ordinal()`**: 선언된 순서(0부터 시작)를 반환
  → 단, **ordinal 값에는 절대 의존하지 말 것.**
  상수 추가/삭제 시 순서가 변하면 호환성이 깨진다.