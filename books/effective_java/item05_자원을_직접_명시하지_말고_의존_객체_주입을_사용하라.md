# ITEM 05. 자원을 직접 명시하지 말고 의존 객체 주입을 사용하라
> 클래스가 내부적으로 **하나 이상의 자원**에 의존하고, 그 자원이 클래스 동작에 영향을 준다면 **싱글턴과
> 정적 유틸리티 클래스는 적합하지 않다**. 대신 **의존객체 주입을 사용**하자.
---
## 의존 객체 주입 패턴
```java
public class SpellChecker {
    private final Lexicon dictionary;

    // 의존 객체를 생성자가 주입받음
    public SpellChecker(Lexicon dictionary) {
        this.dictionary = Objects.requireNonNull(dictionary);
    }

    public boolean isValid(String word) {}

    public List<String> suggestions(String typo) {}
}

```
### 장점
- 클라이언트에게 자원을 받음으로써 여러 자원을 지원할 수 있다.(유연성 UP)
- 사용하는 자원의 **불변을 보장**하여 클라이언트가 안정적으로 클래스를 사용할 수 있게 된다.
### 단점
- 의존 객체가 많아질수록 코드가 어지러워진다. 매번 호출할 때마다 엄청나게 많은 객체를 입력해주어야하기 때문.
      
    → 스프링 프레임워크를 사용하여 해소
---
## 의존 객체 주입 패턴 변형 : 자원 대신 `Supplier<T>` 주입
```java
public class SpellChecker {
    private final Supplier<Lexicon> dictionarySupplier;

    public SpellChecker(Supplier<Lexicon> dictionarySupplier) {
        this.dictionarySupplier = dictionarySupplier;
    }

    public boolean isValid(String word) {
        Lexicon dict = dictionarySupplier.get(); // 필요할 때 꺼내 씀
        // ...
        return true;
    }
}
```
## 장점
- 지연 로딩 할 수 있어 성능에 이득
- 매 쓰레드마다 새로운 자원을 생성해서 줄 수 있음

---
## 책에 소개된 잘못된 예시
### 정적 유틸리티 클래스
```java
public class SpellChecker {
    private static final Lexicon dictionary = new KoreanDictionary();

    private SpellChecker() { } // 인스턴스화 방지

    public static boolean isValid(String word) {}

    public static List<String> suggestions(String typo) {}
}
```
- SpellChecker가 어떤 Lexicon을 쓸지 스스로 결정해버려 다른 사전(EnglishDictionary 등)으로 바꾸려면 코드 수정이 필요함.
- 테스트도 어려운 유연성 낮은 코드 

---
## Spring 프레임워크가 복잡성을 해소한다는 의미
- 처음 읽고 `@Autowired`나 `Lombok`같이 **의존 객체 주입을 사용하는 곳**을 염두한 말인 거라고 추측.
- 하지만 의존 객체 주입 패턴 클래스를 사용하는 **클라이언트의 불편함을 
  DI 컨테이너가 자동으로 인식하여 주입해주기에 해소**된다는 점을 강조한 것으로 보인다.

---
## `@Autowired`는 생성자에 붙이는걸 권장하는 이유
> 이전부터 멤버변수가 아닌 생성자에 `@Autowired`를 사용하는걸 권장한다는 말을 들었는데 이번 챕터와 연관된 것으로 보여 조사

### 의존 객체 주입 방법
#### 멤버변수에 선언 시
1. 기본 생성자를 통해 인스턴스 생성
2. 리플렉션(Reflection)을 이용해서 @Autowired가 붙은 필드를 찾아냄
2. 컨테이너에서 해당 타입에 맞는 빈을 검색
3. Field.setAccessible(true) 후 field.set(orderService, resolvedBean) 호출


#### 생성자에 선언
1. @Autowired가 붙은 생성자를 우선적으로 실행되어 빈을 찾아 주입
2. 즉 객체 생성과 동시에 DI가 완료

### 생성자 주입의 장점
- 생성자 주입는 `final`이 가능하여 불변함을 보장할 수 있고, 
- 자원을 직접 클라이언트가 다룰 수 있어 더 유연한 코드가 되어 테스트에 용의하다
