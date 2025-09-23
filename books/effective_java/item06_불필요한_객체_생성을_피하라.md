# ITEM 06. 불필요한 객체 생성을 피하라
> 객체 생성이 비싸니 최소한으로 만들라는게 아니라, **생성이 무거운 객체는 캐싱해서 사용하는 방안을 고려하자**는 의미이다.
> 
> 요즘 JVM은 작은 객체를 생성하고 회수하는덷 큰 부담이 없고, 
> **방어적 복사가 필요한 상황에서 객체를 재사용했을 때의 피해가, 
> 객체 생성을 반복 실행했을 때의 피해보다 훨신 크다는 걸 염두하자.**  

---
## 생성 비용이 비싼 객체는 재사용하라
대표적으로 Pattern 인스턴스가 있다. Pattern 인스턴스는 입력받은 정규표현식에 해당하는 
유한 상태 머신을 만들기 때문에 인스턴스 생성 비용이 높다.

> **유한 상태머신(Finite State Machine, FSM)과 비용**
> - FSM 자체: 유한한 상태 집합과 전이 규칙을 모델링하는 구조. → 실행은 가볍다. 
> - FSM 생성 비용: 정규식 같은 경우, FSM을 매번 “새로 만들 때” 비용이 크다.
> - 아이템 06의 포인트는 **“FSM을 매번 생성하지 말고, 미리 만들어 재사용하라”**는 것.
> - enum, 싱글턴, 정적 상수 등을 활용.

> FSM 예시
> - 정규식: `Pattern`
> - 포매터: `SimpleDateFormat`, `DateTimeFormatter` (생성 자체는 무겁지만 불변이라 캐싱 필요)
> - 암호화: `Cipher`, `MessageDigest` (Provider 초기화 비용)

---
## 정규식(Pattern) 예시
### 나쁜 예시
```java
static boolean isRomanNumeral(String s) {
    return s.matches("^(?=.)M*(C[MD]|D?C{0,3})"
                        + "(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");
}
```
- String.matches(...)는 내부적으로 Pattern.compile(...)을 호출한다.
- Pattern.compile은 정규식을 FSM(유한 상태 머신)으로 파싱·구축하기 때문에 비용이 크다.
- 따라서 위 메서드는 호출될 때마다 FSM을 새로 생성하는 낭비가 발생한다.

### 권장 예시
```java
private static final Pattern ROMAN = Pattern.compile(
        "^(?=.)M*(C[MD]|D?C{0,3})"
                + "(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");

static boolean isRomanNumeral(String s) {
    return ROMAN.matcher(s).matches();
}
```
- Pattern은 불변(immutable) + 스레드 안전 → 정적 상수로 캐싱 가능.
- FSM을 한 번만 생성하고 재사용하기 때문에 훨씬 효율적이다.

---
## 지연 초기화가 늘 좋을까?
- 인스턴스를 미리 만들어놓고 사용하게 되면, 메서드가 한 번도 호출되지 않을 때 쓸데없이 초기화 된 셈이다
- 이를 방지하기 위해 **지연 초기화를 할 수 있지만 권장하지 않는다**
- **코드를 복잡하게 만드는데, 성능은 크게 개선되지 않을 때가 많기 때문**이다.

---
## 대표적인 잘못된 예시
### 문자열
```java
    // ❌ 매번 새로운 String 인스턴스 생성
    String s = new String("hello");
    
    // ✅ 문자열 리터럴은 재사용됨
    String s = "hello";
```
- "hello"는 상수 풀에 저장되어 JVM 내에서 재사용된다.
- new String(...)을 쓰면 같은 내용이라도 매번 새로운 객체가 생성된다.

### 박싱된 기본 타입
```java
public static void main(String[] args) {
    // ❌ 불필요한 오토박싱 → 매번 객체 생성
    Long sum = 0L;
    for (long i = 0; i < Integer.MAX_VALUE; i++) {
        sum += i; // long → Long 오토박싱
    }

    // ✅ 기본 타입 사용 (더 빠르고 가벼움)
    long sum = 0L;
    for (long i = 0; i < Integer.MAX_VALUE; i++) {
        sum += i;
    }
}
```
- 오토박싱은 눈에 안 띄게 객체 생성 비용을 유발한다.