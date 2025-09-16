# ITEM 1. 생성자 대신 정적 팩터리 메서드를 고려하라.
> 정적 팩터리 메서드는 **생성자 대신 객체를 반환하는 정적 메서드**를 말한다

## 정적 팩토리 메서드 장점
### 1. 이름을 가질 수 있다.
```java
public class Book {
    private String title;
    private String author;

    // 생성자
    public Book(String title, String author) {
        this.title = title;
        this.author = author;
    }

    // 정적 팩토리 메서드
    public static Book of(String title, String author) {
        return new Book(title, author);
    }

    public static Book from(String csv) {
        String[] parts = csv.split(",");
        return new Book(parts[0], parts[1]);
    }
}
```
- 이름만 잘 지으면 반환될 객체의 특성을 쉽게 묘사할 수 있다.
- 요즘 자주 사용되는 정적 팩토리 컨벤션이 존재.
    - `of`: 매개변수를 변환하지 않고 바로 생성하는 정적 메서드
    - `from`: 매개변수를 변환하여 생성하는 정적 메서드

### 2. 호출될 때마다 인스턴스를 새로 생성하지 않아도 된다.
```java
public class Boolean {
    private final boolean value;

    private static final Boolean TRUE = new Boolean(true);
    private static final Boolean FALSE = new Boolean(false);

    private Boolean(boolean value) {
        this.value = value;
    }

    public static Boolean valueOf(boolean b) {
        return b ? TRUE : FALSE;
    }
}
```
- 불변 클래스는 인스턴스를 미리 만들어놓아 재사용하는 식으로 불필요한 객체 생성을 피할 수 있다.
- 즉 인스턴스 통제를 할 수 있다.

> #### 📌 인스턴스 통제 예시
>
> 1. 싱글톤
     >   ```java
>   public class Evis {
>       private static final Evis INSTANCE = new Evis();
>       private Evis() {}
>       public static Evis getInstance() {
>       return INSTANCE;
>   }
>   ```
> 2. Flyweight 패턴
     >   ```java
>        Integer a = Integer.valueOf(100);
>        Integer b = Integer.valueOf(100);
>
>        Integer x = new Integer(100);
>        Integer y = new Integer(100);
>
>        System.out.println(a == b); // true (같은 객체 재사용)
>        System.out.println(x == y); // false (항상 새 객체 생성)
>    ```
     >   - 자주 사용하는 객체를 미리 만들어놓고 재사용하는 패턴

### 3. 반환 타입의 하위 타입 객체를 반환할 수 있다.
### 4. 입력 매개변수에 따라 매번 다른 클래스의 객체를 반환할 수 있다.
```java
public interface Shape {
    void draw();
}

public class Circle implements Shape {
    @Override
    public void draw() {
        System.out.println("Drawing a Circle");
    }
}

public class Square implements Shape {
    @Override
    public void draw() {
        System.out.println("Drawing a Square");
    }
}

public class ShapeFactory {
    // 정적 팩토리 메서드
    public static Shape createShape(String type) {
        switch (type.toLowerCase()) {
            case "circle":
                return new Circle();
            case "square":
                return new Square();
            default:
                throw new IllegalArgumentException("Unknown shape type");
        }
    }
}
```
- 리턴 타입은 인터페이스로 지정하고, 실제로는 인터페이스의 구현체를 리턴함으로서  
  구현체의 API 는 노출시키지 않고 객체를 생성할 수 있다.
- 클라이언트는 인터페이스만 알고 있기 때문에 구현체가 바뀌어도 영향을 받지 않는다.
- Java 8 이후부터는 인터페이스에 default 메서드와 static 메서드를 정의할 수 있기 때문에
  인스턴스화 불가 동반 클래스를 둘 필요가 없다.
    - ex) `java.util.Collections`
- Java 9 이후부터는 private static 메서드도 정의할 수 있기 때문에
  팩토리 메서드의 코드 중복을 줄일 수 있다.

### 5. 정적 팩터리 메서드가 존재하지 않는 시점까지 지연할 수 있다.
```java
public class LazySingleton {
    private static LazySingleton instance;

    private LazySingleton() {}

    public static LazySingleton getInstance() {
        if (instance == null) {
            instance = new LazySingleton(); // 실제로 요청될 때 생성
        }
        return instance;
    }
}
```
- `getInstance()` 메서드가 호출될 때까지 인스턴스가 생성되지 않는다.
- 쓰이지 않으면 아예 만들어지지 않는 **지연 생성**을 할 수 있다
- JDBC를 예로 들음
    - Connection 은 추상화
    - Driver 는 제공자(Provider)
    - DriverManager.getConnection() 은 서비스 접근 API = 정적 팩터리 메서드
    - Driver이 SPI(Service Provider Interface) 를 담당하기에 리플렉션 없이 동작 가능
      ```java
      Connection conn = DriverManager.getConnection("jdbc:mysql://...", "user", "pw");
      ```
      ```
      클라이언트
      |
      |  (Service Access API)
      v
      DriverManager.getConnection(url)  // 정적 팩토리 메서드
      |
      |  (SPI: Driver 인터페이스)
      v
      MySqlDriver / OracleDriver / PostgresDriver
      |
      |  (Service Interface 구현체 반환)
      v
      MySqlConnection implements Connection
  
      ```

## 정적 팩토리 메서드 단점
### 1. 상속을 하려면 public 또는 protected 생성자가 필요하다.
- 생성자를 private 으로 막아야 정적 팩터리를 강제할 수 있는데, 그러면 상속도 불가능해짐
- 하지만 상속이 필요한 클래스는 거의 없으므로 큰 단점은 아니다.

### 2. 정적 팩토리 메서드는 API 문서화에서 드러나지 않음
```java
Set<String> s1 = EnumSet.of(Day.MONDAY);
Set<String> s2 = EnumSet.of(Day.MONDAY, Day.TUESDAY);
```
- s1 은 내부적으로 RegularEnumSet 반환
- s2 는 내부적으로 JumboEnumSet 반환
- 개발자는 코드만 보고는 이걸 알 수 없음 → 디버깅 시점에만 확인 가능