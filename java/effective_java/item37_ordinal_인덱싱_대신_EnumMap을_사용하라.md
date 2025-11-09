# item 37. ordinal 인덱싱 대신 EnumMap을 사용하라
>1. **ordinal을 배열 인덱스로 사용하는 코드는 취약하다.**
> 2. **EnumMap은 타입 안전하고, 명확하며, 유지보수가 쉽다.**
> 3. **EnumSet / EnumMap**은 `enum`을 키로 사용하는 경우 거의 항상 더 나은 선택이다.
> 4. 복잡한 관계(예: enum 쌍 매핑)도 **중첩 EnumMap**으로 표현하라.

## 1. 개요

열거 타입(`enum`)은 상수 집합을 표현하기에 매우 적합하다.
하지만 열거 타입의 **순서값(ordinal)** 을 이용하여 배열이나 리스트의 인덱스로 사용하는 것은 **유지보수성과 안전성** 모두에 문제를 일으킨다.
이를 대체할 수 있는 타입 안전한 컬렉션이 바로 **`EnumMap`** 이다.

---

## 2. 문제점: ordinal을 인덱스로 사용하는 방식

```java
class Plant {
    enum LifeCycle { ANNUAL, PERENNIAL, BIENNIAL }

    final String name;
    final LifeCycle lifeCycle;

    Plant(String name, LifeCycle lifeCycle) {
        this.name = name;
        this.lifeCycle = lifeCycle;
    }

    @Override public String toString() {
        return name;
    }
}
```

아래 코드는 각 생애주기(LifeCycle)별로 식물을 분류하려는 코드이다.

```java
Set<Plant>[] plantsByLifeCycle = 
        (Set<Plant>[]) new Set[Plant.LifeCycle.values().length];

for (int i = 0; i < plantsByLifeCycle.length; i++)
    plantsByLifeCycle[i] = new HashSet<>();

for (Plant p : garden)
    plantsByLifeCycle[p.lifeCycle.ordinal()].add(p);

for (int i = 0; i < plantsByLifeCycle.length; i++)
    System.out.printf("%s: %s%n",
        Plant.LifeCycle.values()[i], plantsByLifeCycle[i]);
```

### 문제점 요약

* **배열은 제네릭을 지원하지 않기 때문에 캐스팅 경고 발생** (`unchecked cast`)
* **타입 안전하지 않다**
  → 인덱스를 잘못 계산하거나 enum의 순서가 바뀌면 엉뚱한 위치에 저장됨
* **가독성이 나쁘다**
  → 코드에서 `ordinal()` 호출이 의미를 알기 어렵다.

---

## 3. 개선: EnumMap 사용

```java
Map<Plant.LifeCycle, Set<Plant>> plantsByLifeCycle =
        new EnumMap<>(Plant.LifeCycle.class);

for (Plant.LifeCycle lc : Plant.LifeCycle.values())
    plantsByLifeCycle.put(lc, new HashSet<>());

for (Plant p : garden)
    plantsByLifeCycle.get(p.lifeCycle).add(p);

System.out.println(plantsByLifeCycle);
```

### 장점

1. **타입 안전성 확보**

    * `LifeCycle` 이외의 키는 사용할 수 없음
2. **가독성 향상**

    * `ordinal()` 없이 의미 있는 키로 접근 가능
3. **내부적으로 효율적**

    * `EnumMap`은 내부적으로 배열로 구현되어 있으므로 성능은 동일하지만 타입 안전성을 보장한다.
4. **제네릭과 완벽하게 호환**

---

## 4. 응용 예시: 열거 타입 쌍의 관계 표현 (Phase 예시)

`Phase` 예시는 **두 열거 값 사이의 전이 관계**를 매핑하는 문제를 다룬다.

- 중첩 Map 사용 목적
    - 상태값(from, to)를 통해 열거형(Transition)을 얻고 싶다! 
```java
enum Phase {
    SOLID, LIQUID, GAS;

    public enum Transition {
        MELT(SOLID, LIQUID),
        FREEZE(LIQUID, SOLID),
        BOIL(LIQUID, GAS),
        CONDENSE(GAS, LIQUID),
        SUBLIME(SOLID, GAS),
        DEPOSIT(GAS, SOLID);

        private final Phase from;
        private final Phase to;

        Transition(Phase from, Phase to) {
            this.from = from;
            this.to = to;
        }

        // 이전 방식: 2차원 배열 + ordinal 사용
        private static final Transition[][] TRANSITIONS = {
            { null, MELT, SUBLIME },
            { FREEZE, null, BOIL },
            { DEPOSIT, CONDENSE, null }
        };

        public static Transition from(Phase from, Phase to) {
            return TRANSITIONS[from.ordinal()][to.ordinal()];
        }
    }
}
```

### 문제점

* ordinal 기반의 배열 인덱싱이라 **타입 안정성이 깨진다.**
* Phase의 상수가 추가되면 **배열 크기와 순서를 모두 수정해야 한다.**
* 열거 타입 변경 시 오류가 컴파일 타임이 아닌 **런타임에 발생**한다.

---

## 5. EnumMap으로 개선한 버전

```java
enum Phase {
    SOLID, LIQUID, GAS;

    public enum Transition {
        MELT(SOLID, LIQUID),
        FREEZE(LIQUID, SOLID),
        BOIL(LIQUID, GAS),
        CONDENSE(GAS, LIQUID),
        SUBLIME(SOLID, GAS),
        DEPOSIT(GAS, SOLID);

        private final Phase from;
        private final Phase to;

        Transition(Phase from, Phase to) {
            this.from = from;
            this.to = to;
        }

        // EnumMap 기반 관계 맵 구성
        private static final Map<Phase, Map<Phase, Transition>> m =
                Arrays.stream(values())
                        .collect(Collectors.groupingBy(
                                t -> t.from, // 1. 그룹화 기준
                                () -> new EnumMap<>(Phase.class), // 1. map 형태 뭐로 할건지
                                Collectors.toMap( // 1. value(여기선 Map)
                                        t -> t.to, // 2. key
                                        t -> t, // 2. value
                                        (x, y) -> y, // 2. merge function
                                        () -> new EnumMap<>(Phase.class)))); // 2. map 형태 뭐로 할건지

        public static Transition from(Phase from, Phase to) {
            return m.get(from).get(to);
        }
    }
}
```

### 개선 효과

* **가독성 및 유지보수성 향상**: Phase가 추가되어도 Transition만 수정하면 됨
* **컴파일타임 타입 안전성 확보**: 키 타입이 `Phase`로 고정
* **배열보다 표현력이 뛰어나고, 의미를 명확히 드러냄**
