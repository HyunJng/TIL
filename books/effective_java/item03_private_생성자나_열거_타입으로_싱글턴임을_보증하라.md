# ITEM 03. private 생성자나 열거 타입으로 싱글턴임을 보증하라
## 싱글턴의 단점
- 생성자가 private 이기에 상속이나 mock 구현이 불가능해 클라이언트를 테스트하기가 어려워진다.
- 단, 리플렉션을 사용하여 private 생성자를 호출이 가능하다. 
  따라서 이런 경우를 방어하기 위해 생성자를 수정하여 두 번째 객체가 생성되려할 때 예외를 던지는 방어로직을 작성하자.

---
## 싱글턴 작성 방법
### 1. `public static final`
```java
public class Logger {
    public static final Logger INSTANCE = new Logger(); // public인 것이 포인트

    private Logger() { }

    public void log(String msg) {
        System.out.println(msg);
    }
}
```
- 간결하고 해당 클래스가 싱글턴임이 API 에 명확히 드러난다.
- 리플렉션을 통한 예외처리는 처리해주어야한다.

### 2. 정적 팩터리 방식
```java
public class Logger {
    private static final Logger INSTANCE = new Logger(); // private인 것이 포인트

    private Logger() { } 

    public static Logger getInstance() {
        return INSTANCE;
    }

    public void log(String msg) {
        System.out.println(msg);
    }
}
```
#### 장점 1. API를 바꾸지 않고도 싱글턴이 아니게 변경할 수 있다.
- Elvis.getInstance() → 싱글턴 반환
- 내부 구현만 바꾸면 매번 새로운 객체 반환도 가능
#### 장점 2. 정적 팩터리를 제네릭 싱글턴 팩터리로 만들 수 있다.
- 한 번만 만들어도 되는 불변 객체(항등 함수, 비교기 등)를 타입별로 안전하게 제공

```java
private static final List EMPTY_LIST = new EmptyList<> ();

@SuppressWarnings("unchecked")
public static final <T> List<T> emptyList() {
  return (List<T>) EMPTY_LIST;
}
```
- 장점 
    - 불필요한 객체 생성 방지 (메모리 절약, 성능 효율)
    - 타입별 안전성 제공 (String, Integer 등에서 안전하게 사용)
- 효용성 
    - 프레임워크/라이브러리 개발에서 자주 쓰임.
    - 예: `Collections.emptyList()`
#### 장점 3. 정적 팩터리 메서드 참조를 공급자로 사용할 수 있다.
```java
Supplier<Elvis> elvisSupplier = Elvis::getInstance;
```
- 정적 팩터리 메서드를 함수형 인터페이스(Supplier<T>)와 바로 연결 가능하다

**이러한 장점이 필요하지 않다면 `public static` 방식을 사용하는 것이 좋다**

> **유의점**
> 
> 1번과 2번 방식으로 만들어진 싱글턴 클래스를 직렬화하려면 단순히 Serializable을 구현한다고 선언하는 것만으로는 부족하다.
> 
> 모든 인스턴스 필드에 transient를 선언하고, readResolve 메서드를 제공해야만 역직렬화시에 새로운 인스턴스가 만들어지는 것을 방지할 수 있다. 만약 이렇게 하지 않으면 초기화해둔 인스턴스가 아닌 다른 인스턴스가 반환된다.

### 3. enum 타입 선언
```java
public enum Elvis {
    INSTANCE;

    public void leaveTheBuilding() { ... }
}
```
- 앞의 예제들에 비해 더 간결하고, 추가 노력 없이 직렬화할 수 있으며, 아주 복잡한 직렬화 상황이나 리플렉션 공격에서도 제2의 인스턴스가 생기는 일을 막아준다. 
- 대부분 상황에서는 원소가 하나뿐인 열거 타입이 싱글턴을 만드는 가장 좋은 방법이다. 
- 단, 만들려는 싱글턴이 Enum 외의 클래스를 상속해야 한다면 이 방법은 사용할 수 없다.