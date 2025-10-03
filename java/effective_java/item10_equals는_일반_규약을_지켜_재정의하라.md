# item10. equals는 일반 규약을 지켜 재정의하라
> - `equals`를 올바르게 재정의하지 않으면 컬렉션(Set, Map)과 같은 자료구조의 동작이 깨지거나, 프로그램이 예기치 않게 동작할 수 있다.
> - equals를 재정의하기는 까다롭기 때문에, 가능하다면 재정의하지 않는 것이 가장 좋다.
> - equals를 직접 정의한다면 5가지 규칙을 반드시 지키자
> - 실무에서는 오픈소스 사용하는 방법도 있다
>   - DTO → @Value (불변, equals/hashCode 자동)
>   - JPA Entity → @EqualsAndHashCode(of = "id") (식별자 기반 동치성)

## `equals`를 재정의하지 않아도 되는 경우
1. **각 인스턴스가 본질적으로 고유하다**
    - 예: Thread, System 같은 객체는 ID 개념이 필요 없다. 인스턴스 고유성이 곧 동치성이다.
2. **인스턴스의 논리적 동치성을 검사할 일이 없다.**
    - 예: Pattern 클래스. 두 정규식이 같은 정규식을 나타내더라도, 굳이 equals로 비교할 필요가 없다.
3. **상위 클래스에서 재정의한 equals가 하위 클래스에서도 딱 들어맞는다.**
    - 예: 대부분의 Set, List, Map 구현체는 이미 상위 클래스에서 정의한 equals가 원하는 대로 동작한다.
4. **클래스가 private이거나 default(package-private)이고 equals메서드를 호출할 일이 없다.**
    - 필요하다면 equals를 호출하면 UnsupportedOperationException을 던지도록 할 수도 있다.

--- 
## `equals`를 재정의해야 하는 경우
- 객체의 식별성(identity) 이 아닌, 논리적 동치성(logical equality) 을 정의하고 싶을 때만 equals를 재정의한다.
- 대표적인 예: Integer, String, BigDecimal 같은 값 클래스(value class).

---
## `equals`를 재정의할때 따라야하는 일반 규약
1. 반사성(Reflexive)
    - x.equals(x)는 항상 true여야 한다. 
    - 자기 자신과는 반드시 같아야 한다.
2. 대칭성(Symmetric)
    - x.equals(y)가 true이면, y.equals(x)도 반드시 true여야 한다.
    - 예: String은 대칭성을 보장하지만, java.sql.Timestamp는 Date와의 비교에서 대칭성을 깬다.
3. 추이성(Transitive)
    - x.equals(y)와 y.equals(z)가 true이면, x.equals(z)도 true여야 한다.
    - 상속 관계에서 equals를 재정의할 때 가장 자주 위배되는 규약이다.
4. 일관성(Consistent)
    - x.equals(y)의 결과는 두 객체의 상태가 변하지 않는 한 여러 번 호출해도 항상 같아야 한다.
    - 랜덤 값, 네트워크 상태 같은 외부 요인에 따라 변하면 안 된다.
5. null-아님(Non-nullity)
    - x.equals(null)은 항상 false여야 한다.
    - 따라서 구현 시 instanceof 검사 또는 Objects.requireNonNull 활용이 권장된다.

---
## `equals` 구현 요령
```java
public final class PhoneNumber {
    private final short areaCode, prefix, lineNum;

    public PhoneNumber(int areaCode, int prefix, int lineNum) {
        this.areaCode = (short) areaCode;
        this.prefix = (short) prefix;
        this.lineNum = (short) lineNum;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;                 // 1. 자기 자신 검사
        if (!(o instanceof PhoneNumber)) return false; // 2. 타입 검사
        PhoneNumber pn = (PhoneNumber) o;
        return pn.lineNum == lineNum
                && pn.prefix == prefix
                && pn.areaCode == areaCode;         // 3. 핵심 필드 비교
    }

    @Override
    public int hashCode() {
        return Objects.hash(areaCode, prefix, lineNum); // equals와 hashCode는 세트
    }
}
```
- == 연산자로 자기 자신과의 비교를 빠르게 처리한다.(성능 향상)
- instanceof로 입력 타입을 검사한다.
    - 자연스럽게 null에 안전하게 동작하도록 한다.
- 적절한 타입으로 형변환 후, 핵심 필드들이 모두 같은지 비교한다.
- equals를 재정의하면 hashCode도 반드시 재정의해야 한다(아이템 11).

---
## 상속보다는 컴포지션을 사용하라
- 상속으로 확장해 새로운 값을 추가하면서 equals 규약을 만족시키는 방법은 존재하지 않는다.
- **상속대신 Composition 을 사용하라**
- 단, 상위 클래스가 구체를 만들지못하는 absract 클래스 같은 경우는 상관 없다.(상위 equlas와 충돌하는 상황이 없기 때문)
### 상속을 통한 잘못된 `equals` 구현
```java
public class Point {
    private final int x;
    private final int y;

    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Point)) return false;
        Point p = (Point) o;
        return p.x == x && p.y == y;
    }
}
```
```java
// Point를 상속한 ColorPoint
public class ColorPoint extends Point {
    private final String color;

    public ColorPoint(int x, int y, String color) {
        super(x, y);
        this.color = color;
    }

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof ColorPoint)) return false;
        ColorPoint cp = (ColorPoint) o;
        return super.equals(o) && cp.color.equals(color);
    }
}
```
- **대칭성 위배**
    ```java
    Point p = new Point(1, 2);
    ColorPoint cp = new ColorPoint(1, 2, "red");
    
    System.out.println(p.equals(cp));   // true  (Point 입장: 좌표만 같으면 된다)
    System.out.println(cp.equals(p));   // false (ColorPoint 입장: 좌표 + 색 모두 같아야 한다)
    ```
- **추이성 위배**
    ```java
    ColorPoint red = new ColorPoint(1, 2, "red");
    ColorPoint blue = new ColorPoint(1, 2, "blue");
    Point p = new Point(1, 2);
    
    // p.equals(red) == true
    // p.equals(blue) == true
    // 하지만 red.equals(blue) == false
    ```
- **getClass() 사용해서 같은 타입만 비교한다면?**
    - 대칭성, 추이성 등 규약은 지켜지지만
    - 리스코프 치환 법칙에 위배된다.

### 컴포지션을 통한 올바른 `equals` 구현
```java
public class Point {
    private final int x;
    private final int y;

    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Point)) return false;
        Point p = (Point) o;
        return p.x == x && p.y == y;
    }

    @Override
    public int hashCode() {
        return 31 * x + y;
    }
}
```
```java
// ColorPoint: Point를 "상속"하지 않고 "포함"한다.
public class ColorPoint {
    private final Point point;
    private final String color;

    public ColorPoint(int x, int y, String color) {
        this.point = new Point(x, y);
        this.color = color;
    }

    // ColorPoint 고유의 동치성 정의
    @Override
    public boolean equals(Object o) {
        if (!(o instanceof ColorPoint)) return false;
        ColorPoint cp = (ColorPoint) o;
        return cp.point.equals(point) && cp.color.equals(color);
    }

    @Override
    public int hashCode() {
        return 31 * point.hashCode() + color.hashCode();
    }

    // Point 뷰 제공 (Point의 equals 규약과 충돌하지 않음)
    public Point asPoint() {
        return point;
    }
}
```
- Point와 ColorPoint의 equals 규약 충돌이 없다.
- ColorPoint는 자신만의 동치성을 정의하고, 필요하면 asPoint()로 Point 뷰(view) 를 노출해 좌표만 비교할 수도 있다.
- 즉, 상속 문제(대칭성/추이성 위반) 를 컴포지션으로 회피한다.

---
## `equals` 구현할 때 TIP
1. 항시 메모리에 존재하는 객체만을 사용하여 결정적 계산만 수행한다.
2. 비교하기 복잡한 필드는 표준형으로 저장해둔 후 표준형끼리 비교하면 경제적이다
    ```java
    public final class CaseInsensitiveString {
        private final String s;
    
        public CaseInsensitiveString(String s) {
            // 표준형으로 변환해서 저장 (대문자 or 소문자 통일)
            this.s = s.toLowerCase();  
        }
    
        @Override
        public boolean equals(Object o) {
            if (!(o instanceof CaseInsensitiveString)) return false;
            CaseInsensitiveString cis = (CaseInsensitiveString) o;
            // 단순 비교만 하면 됨 (복잡한 toLowerCase 호출 불필요)
            return s.equals(cis.s);
        }
    
        @Override
        public int hashCode() {
            return s.hashCode();
        }
    }
    ```
   - equals가 호출될 때마다 toLowerCase()를 실행하면 비용이 크고 불필요.
   - 대신 생성자에서 미리 소문자로 변환(표준형) 해두면, 비교는 String.equals로 끝낼 수 있음.
3. 다를 가능성이 더 크거나 비교하는 비용이 싼 필드부터 비교하면 성능에 좋다.
4. 객체의 논리적 상태와 관련없는 필드는 비교하지 않아야한다.

---
## 구글의 `@AutoValue`
equals, hashCode, toString 같은 반복적이고 오류 발생하기 쉬운 코드를 자동으로 생성해 준다.
```java
import com.google.auto.value.AutoValue;

@AutoValue
abstract class Person {
    abstract String name();
    abstract int age();

    static Person create(String name, int age) {
        return new AutoValue_Person(name, age); // 컴파일 시 생성됨
    }
}
```
```java
Person p1 = Person.create("Alice", 20);
Person p2 = Person.create("Alice", 20);

System.out.println(p1.equals(p2)); // true
System.out.println(p1.hashCode() == p2.hashCode()); // true
System.out.println(p1); // Person{name=Alice, age=20}
```
- 위 코드에서 AutoValue_Person 클래스는 컴파일 시점에 자동 생성된다. 
- 생성된 클래스에는 다음이 자동 구현된다.
  - `equals`, `hashCode`, `toString`, `constructor`
- 사용 제약
    - 불변 클래스만 지원 (필드가 final 성격이어야 함).
    - 상속 지원 X (다른 AutoValue 클래스 상속 불가).
    - 외부 라이브러리라서 프로젝트에 의존성을 추가해야 함.

---
## Lombok의 `@EqualsAndHashCode`
```java
@EqualsAndHashCode
public class Person {
    private String name;
    private int age;
}
```
**컴파일 후**
```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (o == null || getClass() != o.getClass()) return false;
    Person person = (Person) o;
    return age == person.age &&
           Objects.equals(name, person.name);
}

@Override
public int hashCode() {
    return Objects.hash(name, age);
}
```
- 옵션: @EqualsAndHashCode(of = "id") 처럼 특정 필드만 비교 대상에 포함 가능.
- callSuper = true 설정 시 상위 클래스 필드도 비교 가능.
- 하지만 getClass() 비교 때문에, 상속 관계에서 리스코프 치환 원칙 위배가 발생.

--- 
## Lombok의 `@Value`
> `@EqualsAndHashCode` 보다 `@Value` 사용을 권장

값객체를 정의할 때 사용하는 어노테이션으로, 
클래스와 모든 필드를 final로 만들어서 불변 클래스를 자동으로 생성한다.

### 값 객체란
고유한 식별자(ID)가 없고, 속성 값으로만 동치성이 결정되는 객체.
- 예시
  - Money(통화, 금액) → 1,000원과 1,000원은 같음
  - Point(x, y) → (1,2)와 (1,2)는 같음
  - PhoneNumber("010-1234-5678") → 포맷은 달라도 실제 번호가 같으면 같음
- 반대 개념: 엔티티(Entity)
  - 엔티티: 고유한 식별자(예: DB PK, UUID)로 구별됨. 값이 같아도 ID가 다르면 다른 객체.
  - 예: 회원(id=1, 이름=철수) 와 회원(id=2, 이름=철수) → 서로 다른 사람.

### 값 객체의 특징
1. **불변성 (Immutability)**

    - 한 번 생성되면 상태가 변하지 않아야 함.
    - 값이 바뀌면 새로운 객체를 만들어야 함.
    - 예: `new Money(1000, "KRW")` 생성 후 금액을 바꾸는 게 아니라, `new Money(2000, "KRW")` 새 객체를 만듦.

2. **동치성 기준은 속성 값**

   - `equals`/`hashCode`는 모든(또는 핵심) 필드 값에 기반해 구현.

3. **재사용 가능**

   - 불변이므로 캐시, 공유, 멀티스레드 환경에서도 안전.

4. **부작용 방지**

   - 값 객체 자체를 외부에 전달해도 내부 상태가 변하지 않음.

### 코드 예시 (Money)

```java
// Lombok @Value → 불변 + equals/hashCode/toString 자동 생성
@Value
public class Money {
    int amount;
    String currency;
}

public class Example {
    public static void main(String[] args) {
        Money m1 = new Money(1000, "KRW");
        Money m2 = new Money(1000, "KRW");

        System.out.println(m1.equals(m2)); // true (값만 같으면 같음)
    }
}
```

- `m1`과 `m2`는 서로 다른 인스턴스지만, **값이 같으므로 equals → true**.
