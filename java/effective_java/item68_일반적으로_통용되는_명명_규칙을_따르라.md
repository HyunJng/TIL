## item 68. 명명 규칙을 따르라

### 1. 개요

* 자바 플랫폼은 **광범위하고 일관된 명명 규칙(convention)** 을 갖고 있다.
* 이 규칙을 따르면

    * 코드 가독성이 좋아지고
    * 다른 개발자가 코드를 더 쉽게 이해하며
    * API 사용 실수를 줄일 수 있다.
* 명명 규칙은 크게

    1. **철자 규칙(Spelling conventions)**
    2. **문법 규칙(Grammatical conventions)**
       로 나뉜다.

---

## 2. 철자 규칙 (Spelling Conventions)

### 2.1 패키지와 모듈 이름

* 모두 **소문자**
* 단어 사이에 밑줄(_) 사용하지 않음
* 도메인 역순 사용

```java
com.google.common.collect
org.springframework.security
```

* 모듈 이름도 패키지와 동일한 규칙 적용

---

### 2.2 클래스와 인터페이스 이름

* **대문자로 시작**
* **명사 또는 명사구**

```java
String
HashMap
ThreadPoolExecutor
```

* 인터페이스도 클래스와 동일한 규칙 사용

    * 예외적으로 `Runnable`, `Comparable` 처럼 **형용사형**도 허용

---

### 2.3 메서드 이름

* **소문자로 시작**
* **동사 또는 동사구**

```java
run()
compute()
add()
remove()
```

---

### 2.4 필드 이름

* 클래스, 인터페이스 이름과 동일한 규칙
* `static final` 상수는 예외

```java
private int size;
private final int capacity;
```

---

### 2.5 상수 이름 (`static final`)

* **모두 대문자**
* 단어 사이는 `_` 로 구분

```java
public static final int MAX_VALUE = 100;
public static final String DEFAULT_ENCODING = "UTF-8";
```

---

### 2.6 지역 변수 이름

* **소문자로 시작**
* 짧고 의미 있게
* 문맥이 명확한 경우 한 글자도 허용

```java
int i;
String result;
User user;
```

---

### 2.7 타입 매개변수 이름 (제네릭)

* 관례적으로 한 글자 사용

| 이름   | 의미            |
| ---- | ------------- |
| T    | Type          |
| E    | Element       |
| K    | Key           |
| V    | Value         |
| R    | Result        |
| U, V | 두 번째, 세 번째 타입 |

```java
Map<K, V>
List<E>
Function<T, R>
```

---

## 3. 문법 규칙 (Grammatical Conventions)

### 3.1 클래스

* **명사** 사용
* 객체의 개념을 나타냄

```java
User
Order
Repository
```

---

### 3.2 인터페이스

* 클래스처럼 명사 사용 가능
* 또는 **능력/역할을 나타내는 형용사**

```java
Runnable
Comparable
Iterable
```

---

### 3.3 메서드

#### 3.3.1 동작 수행 메서드

* **동사**

```java
save()
delete()
calculate()
```

---

#### 3.3.2 boolean 반환 메서드

* `is`, `has`, `can` 으로 시작

```java
isEmpty()
hasPermission()
canExecute()
```

---

#### 3.3.3 객체 반환 메서드

* 명사 또는 `get` 접두사

```java
size()
length()
getName()
```

---

#### 3.3.4 타입 변환 메서드

* `toType` 형식

```java
toString()
toArray()
```

---

#### 3.3.5 뷰(View) 반환 메서드

* `asType` 형식
* 내부 상태를 바꾸지 않는 **뷰**를 반환

```java
asList()
asMap()
```

---

#### 3.3.6 복사 메서드

* `copy`, `clone` 보다는 **`copyOf`** 선호

```java
copyOf(original)
```

---

### 3.4 팩터리 메서드 이름 관례

* Item 1과 연결되는 내용

```java
from()
of()
valueOf()
instance()
getInstance()
newInstance()
```