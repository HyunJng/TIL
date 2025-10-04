# item 08. finalizer와 cleaner 사용을 피하라

> - 자바에는 객체가 GC로 수거될 때 자동으로 정리 작업을 실행할 수 있는 메커니즘이 존재한다.
>  - `finalizer` (`Object.finalize()` 메서드)
>  - `cleaner` (Java 9부터 제공된 `java.lang.ref.Cleaner`)
> - 겉보기에는 편리해 보이지만, **예측 불가능하고 성능/보안 문제가 심각하여 실무에서는 거의 쓰이지 않는다.**

---

## 어디에 쓰이는가?
### 원래 의도
- **네이티브 자원 해제**: JNI로 할당한 메모리, C 라이브러리 핸들
- **파일/소켓/DB 커넥션 종료**: 외부 자원 정리

```java
class Resource {
    @Override
    protected void finalize() throws Throwable {
        try {
            System.out.println("리소스 해제!");
        } finally {
            super.finalize();
        }
    }
}
````

### Cleaner (Java 9+)

* finalizer 대체재로 나온 기능
* 별도 스레드에서 객체 소멸 시 등록한 정리 작업을 실행

```java
class Resource {
    private static final Cleaner cleaner = Cleaner.create();
    private final Cleaner.Cleanable cleanable;

    Resource() {
        cleanable = cleaner.register(this, () -> {
            System.out.println("리소스 해제!");
        });
    }
}
```

---

## 왜 쓰지 말아야 하는가?

1. **실행 시점 예측 불가**

    * GC가 언제 돌지 몰라서 자원 해제가 늦어질 수 있다.
2. **실행 보장 없음**

    * 프로그램이 종료되면 아예 호출되지 않을 수도 있다.
3. **심각한 성능 저하**

    * finalizer가 붙은 객체는 GC 비용이 **수십 배 증가**한다.
4. **보안 위험**

    * finalizer 공격: 불완전한 객체가 부활해 클래스 불변식을 깨뜨릴 수 있다.

---

## 실무에서의 대안

* 자원 해제는 **항상 명시적으로** 수행해야 한다.
* 표준 방법은 **`AutoCloseable` + `try-with-resources`**.

```java
try (Connection conn = dataSource.getConnection()) {
    // DB 사용
} // 자동으로 close() 호출
```

* finalizer/cleaner는 **라이브러리 내부에서 "백업 안전망"** 정도로만 사용된다.

    * 예: 네이티브 메모리나 OS 핸들 누수 방지용
    * 개발자가 직접 사용하는 경우는 거의 없다.