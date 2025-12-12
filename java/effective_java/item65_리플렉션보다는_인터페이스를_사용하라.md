## item 65. 리플렉션보다는 인터페이스를 사용하라
> * 리플렉션은 강력하지만 위험하다
> * 가능한 한 인터페이스와 다형성으로 대체하라
> * 리플렉션을 쓰더라도, 객체 생성까지만 사용하라

### 1. 리플렉션이란?

* `java.lang.reflect` 패키지를 통해 **런타임에 클래스의 정보(메서드, 필드, 생성자 등)를 조사하고 조작**하는 기능
* 컴파일 시점에 타입을 알지 못해도, 문자열 기반으로 클래스와 메서드를 다룰 수 있음

```java
Class<?> clazz = Class.forName("com.example.Service");
Method method = clazz.getMethod("execute");
method.invoke(clazz.getDeclaredConstructor().newInstance());
```

---

### 2. 리플렉션의 문제점

리플렉션은 강력하지만 **대가가 매우 크다**.

#### 2.1 컴파일 타임 타입 안전성 상실

* 메서드 이름, 파라미터 타입을 문자열로 다룸
* 오타, 시그니처 변경 → **컴파일은 통과하지만 런타임에 실패**

```java
clazz.getMethod("excute"); // 컴파일 OK, 런타임 NoSuchMethodException
```

#### 2.2 가독성과 유지보수성 저하

* 코드가 장황하고 의도가 드러나지 않음
* IDE 자동완성, 리팩토링 지원 거의 불가

#### 2.3 성능 저하

* 리플렉션 호출은 일반 메서드 호출보다 느림
* 접근 검사, 메타데이터 탐색 비용 존재

#### 2.4 예외 처리 복잡성

* `ClassNotFoundException`
* `NoSuchMethodException`
* `IllegalAccessException`
* `InvocationTargetException`
* 정상 로직보다 예외 처리 코드가 더 많아질 수 있음
> But, `ReflectiveOperationException` 으로 한 큐에 처리할 수 있다
---

### 3. 리플렉션이 꼭 필요한 경우

책에서도 **완전히 배제하라고 하지는 않는다**.

* 플러그인 아키텍처
* 런타임에 클래스가 결정되는 경우

단, **이 경우에도 사용 범위를 최소화**해야 한다.

---

### 4. 핵심 대안: 인터페이스 사용

리플렉션이 필요한 대부분의 상황은 **인터페이스로 해결 가능**하다.

#### 4.1 잘못된 설계 (리플렉션 남용)

```java
Object service = Class.forName(className)
    .getDeclaredConstructor()
    .newInstance();

Method m = service.getClass().getMethod("run");
m.invoke(service);
```

* `run()`이 있는지 컴파일 타임에 알 수 없음
* 변경에 매우 취약

#### 4.2 올바른 설계 (인터페이스 활용)

```java
public interface Service {
    void run();
}
```

```java
Service service = serviceLoader.load();
service.run();
```

* 컴파일 타임 타입 체크 가능
* IDE 지원, 리팩토링 안전
* 코드 의도가 명확

---

### 5. 리플렉션을 쓴다면 지켜야 할 원칙

* 리플렉션은 **객체 생성 단계까지만 사용**
* 생성된 객체는 **인터페이스 타입으로 다룬다**

```java
Service service =
    (Service) Class.forName(className)
        .getDeclaredConstructor()
        .newInstance();

service.run(); // 이후는 일반 호출
```

* 리플렉션 코드와 비즈니스 로직을 철저히 분리

---

### 6. 실무에서

* 리플렉션은 **프레임워크 개발자의 도구**
* 애플리케이션 코드에서 리플렉션이 보인다면:

    * 설계 냄새(code smell)일 가능성 높음
* “유연성”을 이유로 리플렉션을 선택하기 전에:

    * 인터페이스 + 다형성으로 해결 가능한지 먼저 검토