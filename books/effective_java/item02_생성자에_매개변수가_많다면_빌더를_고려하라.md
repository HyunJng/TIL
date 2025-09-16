# ITEM 2. 생성자에 매개변수가 많다면 빌더를 고려하라
## Builder 패턴
> 매개변수는 시간이 지날수록 많아질 가능성이 있기에 처음부터 빌더로 시작하는 편이 나을 때가 많다.
- 장점
  - 가독성: 어떤 값을 넣는지 명확히 드러남 (calories(100) vs new NutritionFacts(240,8,100,0,35,27) 차이)
  - 안정성: 불변 객체로 만들 수 있음.
- 단점
    - 구현을 위한 코드가 장황함
    - 생성비용은 크지 않아도 성능에 민감한 상황이라면 문제가 될 수 있다

```java
public class NutritionFacts {
    private final int servingSize;
    private final int servings;
    private final int calories;
    private final int fat;
    private final int sodium;
    private final int carbohydrate;

    public static class Builder {
        // 필수
        private final int servingSize;
        private final int servings;

        // 선택 - 기본값으로 초기화
        private int calories = 0;
        private int fat = 0;
        private int sodium = 0;
        private int carbohydrate = 0;

        public Builder(int servingSize, int servings) {
            this.servingSize = servingSize;
            this.servings = servings;
        }

        public Builder calories(int val) {
            calories = val; return this;
        }

        public Builder fat(int val) {
            fat = val; return this;
        }

        public Builder sodium(int val) {
            sodium = val; return this;
        }

        public Builder carbohydrate(int val) {
            carbohydrate = val; return this;
        }

        public NutritionFacts build() {
            return new NutritionFacts(this);
        }
    }

    private NutritionFacts(Builder builder) {
        servingSize = builder.servingSize;
        servings = builder.servings;
        calories = builder.calories;
        fat = builder.fat;
        sodium = builder.sodium;
        carbohydrate = builder.carbohydrate;
    }
}
```
```java
NutritionFacts cocaCola = new NutritionFacts.Builder(240, 8)
        .calories(100)
        .sodium(35)
        .carbohydrate(27)
        .build();
```
- 필수값은 Builder 생성자에서 받고, 선택값은 메서드 체인 방식으로 설정.
- 최종적으로 build() 메서드로 **불변 객체** 생성.
- `build()`가 호출하는 생성자에서 여러 매개변수의 불변식을 검사하자.

---

## 계층 구조에서 Builder 패턴 사용하는 방법
```java
// 상위 추상 클래스
public abstract class Pizza {
    public enum Topping { HAM, MUSHROOM, ONION, PEPPER, SAUSAGE }
    final Set<Topping> toppings;

    abstract static class Builder<T extends Builder<T>> {
        EnumSet<Topping> toppings = EnumSet.noneOf(Topping.class);

        public T addTopping(Topping topping) {
            toppings.add(Objects.requireNonNull(topping));
            return self();
        }

        abstract Pizza build();

        // 하위 클래스에서 this를 정확히 반환하도록 강제
        protected abstract T self();
    }

    Pizza(Builder<?> builder) {
        toppings = builder.toppings.clone();
    }
}
```
- `abstract static class Builder<T extends Builder<T>>`
    - 하위 클래스에서 빌더 메서드 체인 호출 시 `정확한 타입`을 반환하도록 강제하기 위함 
```java
public class NyPizza extends Pizza {
    public enum Size { SMALL, MEDIUM, LARGE }
    private final Size size;

    public static class Builder extends Pizza.Builder<Builder> {
        private final Size size;

        public Builder(Size size) {
            this.size = Objects.requireNonNull(size);
        }

        @Override
        public NyPizza build() {
            return new NyPizza(this);
        }

        @Override
        protected Builder self() { return this; }
    }

    private NyPizza(Builder builder) {
        super(builder);
        size = builder.size;
    }
}
```
```java
    NyPizza pizza = new NyPizza.Builder(NyPizza.Size.SMALL)
            .addTopping(Pizza.Topping.SAUSAGE)
            .addTopping(Pizza.Topping.ONION)
            .build();
```
- `T extends Builder<T>` 제네릭을 통해 `addTopping`을 호출하더라도 `NyPizza.Builder`을 반환하도록 강제된다.

---

## Lombok 의 @Builder은 어떻게 동작할까?
### 1. `@Builder`
```java
@Builder
public class NutritionFacts {
    private final int servingSize;
    private final int servings;
    private final int calories;
}
```
**롬복이 생성한 코드**
```java
public class NutritionFacts {
    private final int servingSize;
    private final int servings;
    private final int calories;

    private NutritionFacts(int servingSize, int servings, int calories) {
        this.servingSize = servingSize;
        this.servings = servings;
        this.calories = calories;
    }

    // 정적 내부 클래스: NutritionFactsBuilder
    public static class NutritionFactsBuilder {
        private int servingSize;
        private int servings;
        private int calories;

        NutritionFactsBuilder() {}

        public NutritionFactsBuilder servingSize(int servingSize) {
            this.servingSize = servingSize;
            return this;
        }

        public NutritionFactsBuilder servings(int servings) {
            this.servings = servings;
            return this;
        }

        public NutritionFactsBuilder calories(int calories) {
            this.calories = calories;
            return this;
        }

        public NutritionFacts build() {
            return new NutritionFacts(servingSize, servings, calories);
        }
    }

    public static NutritionFactsBuilder builder() {
        return new NutritionFactsBuilder();
    }
}
```
- 위에서 확인한 빌더패턴과 거의 동일
- `builder()`이라는 정적 팩터리 메서드를 사용하여 `new`키워드 없이 실행이 가능하다
- 하나의 정적 inner class만 생성하기에 상속받은 부모 필드는 빌더에서 사용하지 못한다는 제한이 있다.

### 2. `@SuperBuilder`
```java
@SuperBuilder
public class Pizza {
    private final boolean cheese;
}

@SuperBuilder
public class NyPizza extends Pizza {
    private final String size;
}
```
**롬복이 생성한 코드**
```java
public class Pizza {
    private final boolean cheese;

    protected Pizza(PizzaBuilder<?, ?> b) {
        this.cheese = b.cheese;
    }

    public static PizzaBuilder<?, ?> builder() {
        return new PizzaBuilderImpl();
    }

    // 추상 빌더 (부모/자식 공유용)
    public static abstract class PizzaBuilder<C extends Pizza, B extends PizzaBuilder<C, B>> {
        private boolean cheese;

        public B cheese(boolean cheese) {
            this.cheese = cheese;
            return self();
        }

        protected abstract B self();
        public abstract C build();
    }

    // 구체 빌더 구현체
    private static final class PizzaBuilderImpl
            extends PizzaBuilder<Pizza, PizzaBuilderImpl> {
        @Override
        protected PizzaBuilderImpl self() { return this; }
        @Override
        public Pizza build() { return new Pizza(this); }
    }
}

public class NyPizza extends Pizza {
    private final String size;

    protected NyPizza(NyPizzaBuilder<?, ?> b) {
        super(b);
        this.size = b.size;
    }

    public static NyPizzaBuilder<?, ?> builder() {
        return new NyPizzaBuilderImpl();
    }

    public static abstract class NyPizzaBuilder<C extends NyPizza, B extends NyPizzaBuilder<C, B>>
            extends Pizza.PizzaBuilder<C, B> {
        private String size;

        public B size(String size) {
            this.size = size;
            return self();
        }

        @Override
        public abstract C build();
    }

    private static final class NyPizzaBuilderImpl
            extends NyPizzaBuilder<NyPizza, NyPizzaBuilderImpl> {
        @Override
        protected NyPizzaBuilderImpl self() { return this; }
        @Override
        public NyPizza build() { return new NyPizza(this); }
    }
}
```
- 위에서 확인한 **계층관계에서의 빌더 패턴**과 동일
- 어노테이션은 부모와 자식 양쪽에 추가되어야한다.
- “추상 빌더 + 구현체” 구조를 둠으로써 상속 구조에서도 빌더를 안전하게 체이닝할 수 있게 만들어 `@Builder`의 단점 극복

### `@Builder.Default`
```java
@Builder
public class NutritionFacts {
    @Builder.Default
    private int calories = 100;
}
```
**롬복이 생성한 코드**
```java
public static class NutritionFactsBuilder {
    private int calories;
    private boolean calories$set; // 추가됨: 이 필드가 세팅됐는지 여부 표시

    public NutritionFactsBuilder calories(int calories) {
        this.calories = calories;
        this.calories$set = true;
        return this;
    }

    public NutritionFacts build() {
        int calories = this.calories$set ? this.calories : 100; // 기본값 유지
        return new NutritionFacts(calories);
    }
}
```
- 값이 설정되지 않았을 때만 기본값을 적용한다
- 내부적으로 필드명$set 플래그를 만들어서, 사용자가 빌더에서 값을 넣었는지 체크하는 방식으로 동작.
- 이펙티브 자바 책에서 권장한대로 `build()` 내부에서 불변식 검사를 진행하는 것을 확인할 수 있다.
---
## 소개된 BAD CASE
### 자바빈즈패턴(JavaBeans pattern)
```java
    NutritionFacts cocaCola = new NutritionFacts();
    cocaCola.setServingSize(240);
    cocaCola.setServings(8);
    cocaCola.setCalories(100);
    cocaCola.setFat(0);
```
- 매개변수 없는 생성자로 객체를 만든 후 setter로 매개변수의 값 설정하는 방법
- 객체 하나를 만들기 위해 호출하는 메서드가 많고, 객체가 완전히 생성되기 전까지 **일관성이 무너진 상태**가 된다는 단점이 있다.
- 불변한 객체로 만들 수 없어 프로그래머가 안정성을 위한 작업을 해주어야한다.

### 점층적 생성자
```java
public class NutritionFacts {
    private final int servingSize;  // 필수
    private final int servings;     // 필수
    private final int calories;     // 선택
    private final int fat;          // 선택

    public NutritionFacts(int servingSize, int servings) {
        this(servingSize, servings, 0, 0);
    }

    public NutritionFacts(int servingSize, int servings, int calories) {
        this(servingSize, servings, calories, 0);
    }

    public NutritionFacts(int servingSize, int servings, int calories, int fat) {
        this.servingSize = servingSize;
        this.servings = servings;
        this.calories = calories;
        this.fat = fat;
    }
}
```
- 오버로딩 이용한 생성자