# item 33. 타입 안전 이종 컨테이너를 고려하라
> * 제네릭 타입 시스템은 보통 “하나의 타입 매개변수로 단일 타입의 컨테이너”를 다루지만,
  `Class<T>`를 키로 활용하면 **“타입별 객체를 안전하게 관리하는 이종 컨테이너”** 를 만들 수 있다.
> * 이 패턴은 특히 **플러그인, 설정값, 애너테이션, 서비스 레지스트리**처럼
  **타입별로 대표 인스턴스를 보관해야 하는 경우**에 유용하다.

## 1. 개요

제네릭은 보통 **하나의 타입 매개변수로 한 가지 타입의 객체만 다루도록 설계**된다.
예를 들어 `List<String>`은 문자열만, `List<Integer>`는 정수만 담을 수 있다.
하지만 경우에 따라 **서로 다른 타입의 객체를 하나의 컨테이너에 안전하게 저장**해야 할 때가 있다.

이럴 때 사용하는 것이 바로
**타입 안전 이종 컨테이너(typesafe heterogeneous container)** 패턴이다.

---

## 2. 단일 타입 컨테이너의 한계

일반적인 제네릭 컨테이너는 **타입 인자 하나만을 기준으로 타입 안정성을 보장**한다.

```java
Map<String, Integer> map = new HashMap<>();
```

* `Key`는 항상 `String`
* `Value`는 항상 `Integer`

→ `map.put("a", "문자열")`처럼 다른 타입을 넣으면 컴파일 오류가 발생한다.
이 구조에서는 `Key`마다 서로 다른 타입의 값을 저장할 수 없다.

---

## 3. 타입 안전 이종 컨테이너 패턴

이 한계를 해결하기 위해 **Key에 타입 정보를 함께 담는 방식**을 사용한다.
이때 사용하는 것이 **`Class<T>` 객체**다.

```java
public class Favorites {
    private final Map<Class<?>, Object> favorites = new HashMap<>();

    public <T> void putFavorite(Class<T> type, T instance) {
        favorites.put(Objects.requireNonNull(type), instance);
    }

    public <T> T getFavorite(Class<T> type) {
        return type.cast(favorites.get(type));
    }
}
```

이 컨테이너는 내부적으로 `Map<Class<?>, Object>`를 사용하지만,
메서드 시그니처에서 `Class<T>`를 통해 **타입 안전성을 보장**한다.

---

## 4. 사용 예시

```java
Favorites f = new Favorites();

f.putFavorite(String.class, "Java");
f.putFavorite(Integer.class, 123);
f.putFavorite(Class.class, Favorites.class);

String s = f.getFavorite(String.class);
int i = f.getFavorite(Integer.class);
Class<?> c = f.getFavorite(Class.class);
```

* `String.class`를 key로 전달하면 컴파일러는 `T`를 `String`으로 추론한다.
* `getFavorite(String.class)`는 `String` 타입으로 안전하게 반환된다.
* 캐스팅이 내부에서 수행되지만 `Class.cast()`로 타입 검증이 이루어지므로 **런타임 안전성**도 유지된다.

---

## 5. 왜 안전한가?

`Class<T>` 자체가 런타임에 타입 정보를 담고 있기 때문이다.

* `Class<T>#cast()`는 전달된 객체가 실제로 `T` 타입인지 검사한다.
* 잘못된 타입을 넣으면 `ClassCastException`이 발생하며, 이는 컨테이너 내부에서 감지된다.
* 따라서 사용자 입장에서는 컴파일과 런타임 모두 안전하게 동작한다.

즉, **“컴파일 시점에 타입을 보존하면서 여러 타입을 하나의 컨테이너에 저장”** 할 수 있다.

---

## 6. 제약사항

이 패턴에는 다음과 같은 한계가 있다.

1. **클래스 리터럴이 키이므로, 타입별로 오직 하나의 인스턴스만 저장 가능**

   ```java
   f.putFavorite(String.class, "Java");
   f.putFavorite(String.class, "Kotlin"); // 기존 값 덮어씀
   ```

2. **비한정적 제네릭 타입(Class<Set<String>> 등)은 런타임 소거로 인해 구분 불가**

   ```java
   f.putFavorite(new TypeRef<Set<String>>() {}, new HashSet<>());
   ```

   → 이런 경우엔 `TypeToken` 같은 기법이 필요하지만, 책에서는 다루지 않는다.

---

## 7. 실제 적용 예시

자바 표준 라이브러리의 **애너테이션 API**가 이 패턴을 사용한다.

```java
<T extends Annotation> T getAnnotation(Class<T> annotationClass);
```

* `annotationClass`가 키 역할을 한다.
* `Deprecated.class`, `Override.class` 등 애너테이션 타입별로 하나씩 저장된다.
* 이는 결국 **타입 안전 이종 컨테이너**의 형태를 띤다.