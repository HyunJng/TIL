# item 22. 인터페이스는 타입을 정의하는 용도로만 사용하라
> 인터페이스는 클래스가 어떤 역할을 수행할 수 있는지를 나타내는 계약(contract)을 나타내는 용도로만 사용되어야한다.

## 1. 상수 인터페이스란

* 단순히 `public static final` 상수만을 담고 있는 인터페이스를 의미한다.

```java
public interface PhysicalConstants {
    static final double AVOGADROS_NUMBER = 6.022_140_857e23;
    static final double BOLTZMANN_CONSTANT = 1.380_648_52e-23;
    static final double ELECTRON_MASS = 9.109_383_56e-31;
}
```

* 이를 **`implements`하여 상수를 직접 접근**하도록 사용하는 경우가 많았다.

```java
public class Chemistry implements PhysicalConstants {
    ...
    double energy = BOLTZMANN_CONSTANT * temperature;
}
```

---

## 2. 문제점: 인터페이스의 본래 목적 훼손

* 인터페이스는 **“타입을 정의”** 하기 위한 것이다.
  즉, **클래스가 어떤 역할을 수행할 수 있는지를 나타내는 계약(contract)**을 정의해야 한다.
* 그러나 상수 인터페이스를 구현하면,

    * 해당 클래스의 **공개 API에 불필요한 상수가 노출**된다.
    * 그 클래스가 **그 상수와 아무 상관이 없어도** `implements`로 인해 “그 상수를 가진 타입”처럼 보인다.
* 이는 **구현 세부사항이 외부 API에 새어 나가는 것(leak)** 으로,
  향후 리팩터링 시 **하위 호환성을 깨뜨리는 위험**까지 생긴다.

> 즉, "상수 인터페이스는 **타입을 정의하지 않고 구현 세부사항을 노출** 하기 때문에 해롭다."

---

## 3. 대안 1: 유틸리티 클래스 사용

* **상수를 모아두고자 할 때**는 단순히 `final` 클래스로 정의하라.
* 생성자를 `private`으로 막아 **인스턴스화 및 상속을 방지**한다.

```java
public final class PhysicalConstants {
    private PhysicalConstants() {} // 인스턴스화 방지

    public static final double AVOGADROS_NUMBER = 6.022_140_857e23;
    public static final double BOLTZMANN_CONSTANT = 1.380_648_52e-23;
    public static final double ELECTRON_MASS = 9.109_383_56e-31;
}
```

* 사용 시에는 클래스명을 통해 명시적으로 접근한다.

```java
double atoms = count / PhysicalConstants.AVOGADROS_NUMBER;
```

* 이 방식은 **Item 4 (인스턴스화를 막는 유틸리티 클래스)**와 동일한 패턴이며,
  API 명세가 명확하고 구현 누출이 없다.

---

## 4. 대안 2: enum 타입 사용

* 상수들이 **논리적으로 하나의 의미 단위로 묶이는 경우**,
  단순 유틸리티 클래스보다 `enum`이 더 자연스럽다.

```java
public enum Operation {
    PLUS("+")  { public double apply(double x, double y) { return x + y; } },
    MINUS("-") { public double apply(double x, double y) { return x - y; } };

    private final String symbol;
    Operation(String symbol) { this.symbol = symbol; }

    public abstract double apply(double x, double y);
}
```

* 각 상수가 **고유의 의미와 동작을 함께 가질 수 있고**,
  컴파일타임 타입 안정성도 보장된다.