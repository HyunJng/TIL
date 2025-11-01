# item 25. 톱레벨 클래스는 한 파일에 하나만 담으라
> - 한 소스 파일에 여러 톱레벨 클래스를 선언하지 마라.
> - 예측 불가능한 동작을 막고, 유지보수성을 높일 수 있다.

## 1. 주제

* **톱레벨 클래스(top-level class)** 는 **소스 파일 하나당 하나만 선언**해야 한다.
* 같은 파일 안에 여러 톱레벨 클래스를 두면 **예측 불가능한 동작**이 발생할 수 있다.

---

## 2. 왜 문제인가?

Java 컴파일러는 `.java` 파일 단위로 컴파일하되,
**클래스 이름과 파일 이름이 일치하지 않아도 컴파일이 가능**하다.

예를 들어:

```java
// Main.java
class Utensil {
    static final String NAME = "pan";
}
class Dessert {
    static final String NAME = "cake";
}
```

그리고 다음과 같이 또 다른 파일이 있다고 해보자.

```java
// Dessert.java
class Utensil {
    static final String NAME = "pot";
}
class Dessert {
    static final String NAME = "pie";
}
public class Main {
    public static void main(String[] args) {
        System.out.println(Utensil.NAME + " " + Dessert.NAME);
    }
}
```

---

## 3. 어떤 일이 벌어지나?

* **컴파일 순서에 따라 결과가 달라진다.**
* `javac Main.java`로만 컴파일하면 `"pan cake"`이 출력될 수도 있고,
  `javac Dessert.java`로 컴파일하면 `"pot pie"`가 출력될 수도 있다.
* 즉, **같은 코드라도 컴파일 명령에 따라 다른 결과**가 나온다.

---

## 4. 권장 방식

* **톱레벨 클래스는 항상 하나의 파일에만 정의**하라.
  파일 이름도 클래스 이름과 동일하게 맞춘다.

```java
// Utensil.java
class Utensil {
    static final String NAME = "pan";
}

// Dessert.java
class Dessert {
    static final String NAME = "cake";
}
```

* 여러 클래스가 긴밀히 연결되어 있고 외부에서 접근할 필요가 없다면,
  **private 정적 중첩 클래스(private static nested class)**로 선언한다.

```java
public class Kitchen {
    private static class Utensil {
        static final String NAME = "pan";
    }
    private static class Dessert {
        static final String NAME = "cake";
    }
}
```