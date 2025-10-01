# Spring에서 `@Value`와 멤버변수 정의 순서

## 1. 자바 객체 생성 순서

1. **클래스 로딩 & 인스턴스 생성 (`new`)**
2. **멤버변수 기본값 할당**
3. **명시적 초기화 값 적용**

   ```java
   private String status = "init"; // 이 값이 먼저 들어간다
   ```
4. **생성자 실행**

---

## 2. 스프링 빈 초기화 과정

1. **의존성 주입 (Dependency Injection)**

    * `@Autowired`, `@Value`, `@Inject` 적용
    * **생성자 이후에 실행됨**
2. **빈 후처리기 (BeanPostProcessor)** 실행

    * AOP, 프록시 적용 등
3. **초기화 콜백**

    * `@PostConstruct`, `InitializingBean.afterPropertiesSet()` 호출
    * 이 시점에는 `@Value` 값이 이미 주입 완료됨

---

## 3. 실행 예시

```java
@Component
public class TestBean {

    @Value("${app.name:defaultApp}")
    private String appName;

    private String status = "init";

    public TestBean() {
        System.out.println("생성자 실행 - appName=" + appName + ", status=" + status);
    }

    @PostConstruct
    public void init() {
        System.out.println("PostConstruct 실행 - appName=" + appName + ", status=" + status);
    }
}
```

**실행 결과**

```
생성자 실행 - appName=null, status=init
PostConstruct 실행 - appName=실제프로퍼티값, status=init
```

---

## 4. 핵심 정리

* 멤버변수 초기화 → 생성자 실행 → `@Value` 주입 → `@PostConstruct`
* 생성자에서는 `@Value` 값이 아직 없을 수 있다.
* `@Value` 주입된 값을 안전하게 사용하려면 `@PostConstruct` 이후 활용하는 게 좋다.