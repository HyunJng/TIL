# item 39. 명명 패턴보다 애너테이션을 사용하라
> 메서드 이름 규칙(명명 패턴)으로 메타데이터를 표현하지 말고 애너테이션을 써라

## 1. 명명패턴으로 메타데이터 표시한 예시
* 예전: `public static void test1()` 처럼 이름에 `test`를 붙여 테스트 메서드임을 나타냄.
* 문제: 오타(`tset`), 시그니처 규칙 누락, 의미 중복 등 컴파일러가 전혀 도와주지 못함.
* 애너테이션은 **컴파일러가 검증 가능** 하고, 리플렉션으로 깔끔하게 읽어올 수 있으니, 테스트 프레임워크처럼 “메타데이터 기반 도구”에 적합하다.

---

## 2. 애너테이션 정의 기본 형태

### 2.1. 선언 문법

```java
import java.lang.annotation.*;

@Retention(RetentionPolicy.RUNTIME)   // 언제까지 유지할지
@Target(ElementType.METHOD)          // 어디에 붙일 수 있을지
public @interface Test {
}
```

* `@interface` 키워드로 정의한다.
* 보통 최소 다음 두 메타 애너테이션을 거의 필수로 붙인다.

    * `@Retention(RetentionPolicy.RUNTIME)`

        * 런타임에도 유지되도록 해야 리플렉션으로 읽을 수 있다.
    * `@Target(ElementType.METHOD)`

        * 메서드에만 붙일 수 있도록 제한 (클래스, 필드, 파라미터 등은 각각 대응되는 상수 사용)

### 2.2. 요소(element) 정의 규칙

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface ExceptionTest {
    Class<? extends Throwable> value();    // 요소 하나짜리 예시
}
```

* 애너테이션 안의 “메서드처럼 보이는 것들”이 **요소(element)** 이다.
* 요소의 반환 타입에는 다음만 허용된다.

    * 기본 타입(primitive)
    * `String`
    * `Class` (또는 `Class<?>`, `Class<? extends Something>`)
    * `enum` 타입
    * 다른 애너테이션 타입
    * 위 타입들의 1차원 배열
* 요소에 `default` 값을 줄 수 있다.

  ```java
  int retry() default 1;
  ```
* 매개변수는 안 된다.

  ```java
  // 이런 건 불가
  // public int retry(int x);  // X
  ```

---

## 3. 반복 가능(Repeatable) 애너테이션 정의

아이템 39에서 핵심 예시는 `@ExceptionTest`를 **여러 번 붙일 수 있게** 만드는 부분이다.

### 3.1. 컨테이너 애너테이션 정의

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface ExceptionTestContainer {
    ExceptionTest[] value();
}
```

### 3.2. 반복 가능 애너테이션에 @Repeatable 추가

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@Repeatable(ExceptionTestContainer.class)
public @interface ExceptionTest {
    Class<? extends Throwable> value();
}
```

* `@Repeatable(컨테이너클래스.class)` 를 붙이면, **같은 애너테이션을 여러 번 선언**할 수 있다.
* 자바 컴파일러는 내부적으로

    * 여러 개의 `@ExceptionTest`를 `@ExceptionTestContainer` 하나에 담아서 저장해둔다.
* 하지만 리플렉션으로 읽어올 때는 `getAnnotationsByType(ExceptionTest.class)`를 사용하면

    * 컨테이너 내부의 것까지 자동으로 “펼쳐서” `ExceptionTest[]` 배열로 돌려준다

사용 예:

```java
@ExceptionTest(ArithmeticException.class)
@ExceptionTest(NullPointerException.class)
public static void doubleFail() { /* ... */ }
```

---

## 4. 리플렉션으로 애너테이션 읽기

### 4.1. isAnnotationPresent: “이 애너테이션이 붙어있냐?”

```java
import java.lang.reflect.Method;

public class TestRunner {
    public static void main(String[] args) throws Exception {
        int tests = 0;
        int passed = 0;

        Class<?> testClass = Class.forName("Sample");

        for (Method m : testClass.getDeclaredMethods()) {
            // 1) 애너테이션 존재 여부 확인
            if (m.isAnnotationPresent(Test.class)) {
                tests++;
                try {
                    m.invoke(null);    // static 메서드 호출
                    passed++;
                } catch (Throwable wrappedExc) {
                    Throwable exc = wrappedExc.getCause();
                    System.out.println(m + " 실패: " + exc);
                }
            }
        }

        System.out.printf("성공: %d, 실패: %d%n", passed, tests - passed);
    }
}
```

* `m.isAnnotationPresent(Test.class)`

    * 해당 메서드에 `@Test`가 붙어있으면 `true`, 아니면 `false`.
* 주 사용 용도:

    * **“필터” 역할**: 특정 애너테이션이 붙은 메서드만 골라서 처리.
    * `getAnnotation(Test.class) != null` 과 기능상 비슷하지만 가독성이 더 좋다.

### 4.2. getAnnotationsByType: 반복 가능 애너테이션 처리 핵심

반복 가능 애너테이션(`@Repeatable`)을 제대로 쓰려면, 항상 **`getAnnotationsByType`를 써라**가 아이템 39의 메시지다.

예시: `@ExceptionTest`가 하나 또는 여러 개 붙어 있을 수 있는 상황

```java
public class RepeatableTestRunner {
    public static void main(String[] args) throws Exception {
        Class<?> testClass = Class.forName("Sample3");

        for (Method m : testClass.getDeclaredMethods()) {
            // 1) 해당 메서드에 붙은 ExceptionTest를 전부 가져온다.
            ExceptionTest[] excTests =
                    m.getAnnotationsByType(ExceptionTest.class);

            if (excTests.length > 0) {
                System.out.println("테스트 메서드: " + m.getName());
                runExceptionTests(m, excTests);
            }
        }
    }

    private static void runExceptionTests(Method m, ExceptionTest[] excTests) {
        int passed = 0;

        for (ExceptionTest excTest : excTests) {
            try {
                m.invoke(null);
                // 예외가 안났으면 실패
                System.out.printf("  %s 실패 (예외 안 발생)%n", m);
            } catch (Throwable wrappedExc) {
                Throwable exc = wrappedExc.getCause();
                Class<? extends Throwable> expected = excTest.value();
                if (expected.isInstance(exc)) {
                    passed++;
                    System.out.printf("  %s 성공 (%s 발생)%n",
                            m, exc.getClass().getSimpleName());
                } else {
                    System.out.printf("  %s 실패 (기대: %s, 실제: %s)%n",
                            m, expected.getName(), exc.getClass().getName());
                }
            }
        }

        System.out.printf("테스트 결과: %d/%d 성공%n", passed, excTests.length);
    }
}
```

* `m.getAnnotationsByType(ExceptionTest.class)`

    * 해당 메서드에

        * `@ExceptionTest`가 딱 한 번 붙어 있어도,
        * 여러 번 붙어 있어도,
        * 또는 컨테이너 `@ExceptionTestContainer`로만 붙어 있어도,
    * **전부 `ExceptionTest[]` 배열로 반환**한다.
* 만약 `getAnnotation(ExceptionTest.class)`를 썼다면?

    * 여러 번 붙어있더라도 하나만 가져오거나, 아예 `null`을 받을 수 있음.
    * `@Repeatable` 애너테이션을 제대로 활용하지 못하게 된다.
* 그래서 **반복 가능 애너테이션 사용 시에는 항상 `getAnnotationsByType` 사용**이 원칙.

참고로
* `getAnnotation(A.class)`

    * A가 `@Repeatable`이든 아니든, **첫 번째 것 또는 A가 직접 붙어있는 경우 하나만 반환**.
* `getAnnotations(A.class)`

    * 현재 타입과 상위 타입(상속 계층)까지 포함해서 가져오는지 여부 등 세부 동작이 헷갈릴 수 있다.
* 아이템 39는 “헷갈리지 말고 `getAnnotationsByType`을 쓰라”고 딱 짚고 넘어간다.