# item 07. 다 쓴 객체 참조를 해제해라
> - 논리적으로 죽은 객체가 존재한다면, null 처리를 해주어 GC가 발견할 수 있는 조치를 해주자.
> - 최대한 작은 유효범위에 존재하도록 설계를 하자.

## 메모리 누수 예시
- Java는 GC이 다 쓴 메모리를 회수해가기에 메모리 관리에 신경을 "덜"써도 된다
- **논리적인 죽은 객체**가 존재의 경우 GC가 인식할 수 없어 메모리 회수를 할 수 없다.
- 즉, **메모리 누수**가 발생한다는 뜻이다.
```java
 public class Stack {
    private Object[] elements;
    private int size = 0;

    // 나머지 생략
    public Object pop() {
        if (size == 0) {
            throw new EmptyStackException();
        }
        return elements[--size]; // 메모리 누수 발생
    }
}
```
- Stack 클래스이다.
- 스택이 커졌다가 줄어들었을 때, 더이상 사용하지 않는 객체가 존재하지만 GC는 회수하지 않는다.
- 따라서 GC가 인식할 수 있도록 null처리를 해주어야한다.

```java
 public class Stack {
    private Object[] elements;
    private int size = 0;

    public Object pop() {
        if (size == 0) {
            throw new EmptyStackException();
        }
        Object result = elements[--size];
        elements[size] = null; // GC가 인식하도록 null 처리
        return result;
    }
}
```
---
## null 대입보단, 유효범위를 고려하자
- 일부러 null을 대입하기보다는, 변수 자체가 더 이상 쓰이지 않게 만드는 게 가장 자연스럽고 안전하다
- 변수의 범위가 최소가 되도록 해야한다.
    - 이후 item 57에서 자세하게 설명될 예정

### 지역변수 처리가 Best
```java
public void process() {
    Object obj = new Object(); 
    // obj 사용
}
// 여기서 메서드가 끝나면 obj는 유효범위(scope) 밖으로 밀려나서 GC 대상이 됨
```
### 클래스 멤버변수는 명시적 null 처리 필요
```java
public class MemoryLeakExample {
    private List<Object> cache = new ArrayList<>();

    public void add() {
        Object obj = new Object();
        cache.add(obj); // 계속 참조 유지됨
    }
}
```
---
## WeakReference 를 활용하는 방안도 제시
> 그러나 WeakReference에 공부할수록 사용하지 않는 편이 낫겠다고 생각했다.
### Java 의 참조 수준
Java의 **참조(Reference)** 는 크게 네 가지 수준으로 나뉜다.

1. **Strong Reference (강한 참조)** 
    - 우리가 평소에 쓰는 일반적인 참조.
    ```java
    String s = new String("hello");
    ```
   → 이 변수 `s`가 존재하는 한 `"hello"` 객체는 GC(Garbage Collector) 대상이 되지 않는다.

2. **Soft Reference (소프트 참조)**
    - 메모리가 부족할 때만 GC가 수거한다. 캐시 구현에 종종 쓰인다.

3. **Weak Reference (약한 참조)**
    - GC가 객체를 발견하는 순간 바로 수거한다.
    - 즉, **참조만으로는 객체 생존이 보장되지 않는다.**
   ```java
   import java.lang.ref.WeakReference;

   public class Main {
       public static void main(String[] args) {
           String strong = new String("Hello");
           WeakReference<String> weak = new WeakReference<>(strong);

           strong = null; // 강한 참조 제거

           System.gc(); // GC 힌트

           String value = weak.get(); // null 반환할 수 있다
           System.out.println("WeakReference 값: " + value);
       }
   }
   ```
   → GC 실행 시 `weak.get()` 은 `null`을 반환할 수 있다.

4. **Phantom Reference (팬텀 참조)**
   - 객체가 소멸되기 직전에 어떤 동작을 수행하기 위해 사용한다. (예: 직접적인 자원 해제)

### WeakReference의 특징

* **GC에 의해 쉽게 회수**된다. (SoftReference와 달리 메모리 부족 여부와 무관)
* `get()` 메서드로 객체에 접근할 수 있지만, 언제든 `null`이 될 수 있으므로 반드시 `null` 체크 필요.
* 자주 쓰이는 곳:

    * **캐시(Cache)**: 자주 쓰지만 메모리가 부족하면 버려도 되는 데이터
    * **WeakHashMap**: key를 `WeakReference`로 들고 있어서 key가 강한 참조에서 사라지면 자동 제거되는 Map
    * **리스너 패턴**: 이벤트 리스너를 등록했지만 강하게 붙잡고 싶지 않을 때 (메모리 누수 방지)

### WeakHashMap

```java
import java.util.Map;
import java.util.WeakHashMap;

public class WeakHashMapExample {
    public static void main(String[] args) {
        Map<String, String> map = new WeakHashMap<>();
        String key = new String("key");
        map.put(key, "Hello WeakHashMap");

        System.out.println("Before GC: " + map);

        key = null; // 강한 참조 제거
        System.gc();

        // WeakReference라서 GC 이후 자동 제거될 수 있음
        System.out.println("After GC: " + map);
    }
}
```
- 실행 결과
    ```shell
    Before GC: {key=Hello WeakHashMap}
    After GC: {}
    ```
  
### WeakReference 를 실무에서 사용하지 말아야겠다고 생각한 이유
> 책에서는 캐시나 리스너, 콜백의 예시를 주었지만, 
> 캐시에 TTL 적용하는 것처럼 정책적인 부분으로 관리하는 편이 낫다고 결론지었다.
1. **생존이 GC에 전적으로 좌우됨 (비결정성)**
- `WeakHashMap`은 GC에 의해 존재가 좌우되므로 예측하기가 어렵고, 장애분석이 어렵다.
- 아니면 개발자가 강한 참조가 존재하는지 매번 주시하고 있어야한다.

2. **성능 오버헤드 존재**
- 약참조 + `ReferenceQueue` 관리 비용, 엔트리 청소( expunge ) 비용이 존재.

3. **기능적으로 ‘필수 데이터’에는 부적합**
- 구성/설정, 세션 정보, 주문/결제 같은 정합성이 중요한 데이터를 고려하는 경우가 더 많다.
  이런 경우 GC 상황에 따라 사라지면 안되기에  `HashMap`/`ConcurrentHashMap` 사용이 적합하다. 

5. **동시성/운영 기능 부족**
- `WeakHashMap`은 스레드 안전하지 않다고 한다.