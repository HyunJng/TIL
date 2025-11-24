# item 43. 람다보다는 메서드 참조를 사용하라.
> 주제는 제목 그대로이고, 메서드 참조(::)에 대해 알아보았다.

## 1. 개요
자바의 메서드 참조(`::`)는 람다식을 더 간결하게 표현하기 위한 키워드이다.

## 2. 수신 객체(receiver)란?

* 메서드 호출을 “당하는” 객체
* 즉 `obj.method(args...)`에서 **obj가 수신 객체**
* 정적 메서드는 수신 객체가 존재하지 않음

## 3. 메서드 참조의 네 가지 유형

### 3.1. 정적 메서드 참조

형태:

```
ClassName::staticMethod
```

특징:

* 수신 객체 없음
* 함수형 인터페이스의 인수는 그대로 static 메서드의 인수로 전달됨

변환 규칙:

```
(args...) -> ClassName.staticMethod(args...)
```

예:

```
Function<String, Integer> f = Integer::parseInt;
// (x) -> Integer.parseInt(x)
```

### 3.2. 한정적(bound) 인스턴스 메서드 참조

형태:

```
instance::instanceMethod
```

특징:

* 이미 특정 객체가 수신 객체로 “고정”되어 있음
* 함수형 인터페이스의 모든 파라미터는 해당 메서드의 인수로 전달됨

변환 규칙:

```
(args...) -> instance.instanceMethod(args...)
```

예:

```
String s = "hello";
Supplier<Integer> sup = s::length;
// () -> s.length()
```

### 3.3. 비한정적(unbound) 인스턴스 메서드 참조

형태:

```
ClassName::instanceMethod
```

특징:

* 수신 객체가 아직 결정되지 않음
* 함수형 인터페이스의 **첫 번째 매개변수가 수신 객체로 사용됨**
* 나머지 매개변수는 메서드 인수로 전달됨

핵심 변환 규칙:

```
(obj, args...) -> obj.instanceMethod(args...)
```

예:

```
Function<String, Integer> f = String::length;
// (s) -> s.length()
```

또 다른 예:

```
BiFunction<String, String, Boolean> bf = String::startsWith;
// (s, prefix) -> s.startsWith(prefix)
```

### 3.4. 생성자 참조

형태:

```
ClassName::new
```

수신 객체 없음
함수형 인터페이스의 파라미터는 그대로 생성자 인수로 사용됨.

## 4. 정적 vs 비한정적 메서드 참조 차이

| 구분    | 정적 메서드 참조                          | 비한정적 인스턴스 메서드 참조                          |
| ----- | ---------------------------------- | ----------------------------------------- |
| 예     | `Integer::parseInt`                | `String::length`                          |
| 수신 객체 | 없음                                 | 첫 번째 인자가 수신 객체                            |
| 변환 형태 | `(args) -> ClassName.static(args)` | `(obj, args) -> obj.instanceMethod(args)` |
| 의미    | “이 static 메서드를 직접 호출”              | “ClassName 타입 객체의 인스턴스 메서드를 나중에 호출”       |

## 5. 수신객체가 있는지 없는지 어떻게 구분하는지?

자바 컴파일러는 메서드 참조를 보고:

* 메서드 이름을 가진 정적 메서드가 있는 경우 → 정적 메서드 참조
* 없다면 → 인스턴스 메서드로 간주 
* 그리고 함수형 인터페이스의 첫 번째 매개변수를 수신 객체로 매핑

즉, :: 자체가 정적/인스턴스 메서드를 결정하지 않음.
"그 이름의 메서드가 정적 메서드인지"가 기준