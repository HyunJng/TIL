# ITEM 04.인스턴스화를 막으려거든 private 생성자를 사용하라
## 이번장 핵심
- utils 처럼 정적인 메서드와 정적인 필드만이 담긴 클래스처럼 인스턴스화 의도하지 않는 클래스는 
**`interface`나 `abstract class`로 만들지 말고 `private`생성자를 두자**. 
- 추상클래스나 인스턴스로 만든느 것은 인스턴스화를 막을 수 없으며, 
**사용자가 상속해서 사용하라는 뜻으로 오해할 수 있기 때문**이다.
```java
public class UtilityClass {
    // 인스턴스화 방지용
    private UtilityClass() {
        throw new AssertionError();
    }
}
```
- 꼭 `AssertionsError`를 던질 필요는 없지만 
생성자 내부에서 오류를 던지면 클래스 안에서라도 생성자를 호출하지 않도록 보장해준다.
- 그다지 직관적이지 않기에 주석 처리해주는 것을 권장한다.