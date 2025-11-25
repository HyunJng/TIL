# item 46. 스트림에서는 부작용 없는 함수를 사용하라
> - 스트림 철학은 계산을 일렬의 변환으로 구성해서 각 단계마다 이전 단계의 결과를 받아 처리하는 순수 함수여야한다는 것이다.
> - 순수함수는 **오직 입력만이 결과에 영향을 주는 함수**를 말한다.
> - 따라서 종단연산을 구현할 때 외부의 상태를 변환시키는 일을 하지 말자
>   - `forEach()` : 수행한 연산 결과를 보여주는 일을 하면 안된다.
> - 종단연산을 구현할 때, 수집기에 대한 예시가 많았는데, 이를 정리해봤다. 
---

## 1. Collectors에서 가장 중요한 3가지: toMap, groupingBy, joining

### 1.1 toMap – Map 생성 및 병합 로직 지정

#### 1.1.1 기본 형태

* 중복 키 발생 시 `IllegalStateException` 발생.

```
toMap(keyMapper, valueMapper)
```

#### 1.1.2 mergeFunction 제공(3인수)

* 중복 키 발생 시 어떻게 값을 병합할지 지정.
* 타입: `BinaryOperator<V>`

```
toMap(keyMapper, valueMapper, mergeFunction)
```

#### 1.1.3 Map 구현체 지정(4인수)

```
toMap(keyMapper, valueMapper, mergeFunction, mapSupplier)
```

* `EnumMap`, `LinkedHashMap`, `TreeMap` 등 원하는 구현체 지정 가능.

#### 1.1.4 toMap 사용 팁

* 키 중복 가능성이 있다면 반드시 mergeFunction을 제공한다.
* 고유한 Map 구현체가 필요하면 4인수 버전을 사용한다.
* Enum 키면 `EnumMap`을 mapSupplier로 사용해 성능 및 안전성 향상.

---

### 1.2 groupingBy – 분류 + 하위 Collector 조합

#### 1.2.1 기본 형태

```
groupingBy(classifier)
```

* 기본 반환: `Map<K, List<T>>`

#### 1.2.2 downstream Collector 적용
* 다운스트림은 해당 카테고리의 모든 원소를 담은 스트림으로부터 값을 생성하는 역할을 한다.
* 쉽게 말해 `Map<K, List<T>>`에서 `List<T>`말고 다른 컬렉션 만들고 싶을때 지정  
```
groupingBy(classifier, downstream)
```

* 그룹별로 추가적인 수집 가능

    * `counting()`
    * `summingInt()`
    * `maxBy()`
    * `mapping()`
    * `joining()` 등

예: 그룹별 개수

```
groupingBy(Type::of, counting())
```

예: 그룹별 이름 이어붙이기

```
groupingBy(
    Product::getCategory,
    mapping(Product::getName, joining(", "))
)
```

#### 1.2.3 Map 구현체 지정

```
groupingBy(classifier, mapFactory, downstream)
```

* 순서 유지: `LinkedHashMap`
* enum: `EnumMap`
* 병렬 환경: `groupingByConcurrent`

#### 1.2.4 groupingBy 사용 팁

* groupingBy의 진짜 힘은 downstream Collector 조합이다.
* 기존 루프 기반 분류 로직을 깔끔하게 대체한다.
* 너무 복잡한 조합은 사용하지 말고 명시적 루프가 더 나을 수 있다.

---

### 1.3 joining – 문자열 연결 Collector

#### 1.3.1 기본

```
joining()
```

* 구분자 없이 문자열 그대로 연결

#### 1.3.2 구분자 제공

```
joining(", ")
```

#### 1.3.3 prefix, suffix 제공

```
joining(", ", "[", "]")
```

#### 1.3.4 비 String 스트림에서 사용

```
mapping(Object::toString, joining(", "))
```

---

## 2. 그 외 기타 Collectors

### 2.1 컬렉션 수집

* `toList()`
* `toSet()`
* `toCollection(collectionFactory)`

### 2.2 숫자 계산 Collector

* `counting()`
* `summingInt`, `summingLong`, `summingDouble`
* `averagingInt`, `averagingLong`, `averagingDouble`
* `summarizingInt`, `summarizingLong`, `summarizingDouble`

### 2.3 최대/최소/감소 연산

* `maxBy(comparator)`
* `minBy(comparator)`
* `reducing(identity, mapper, operator)`

### 2.4 분할

* `partitioningBy(predicate)`
  `Map<Boolean, List<T>>` 반환

### 2.5 후처리 Collector

* `collectingAndThen(downstream, finisher)`
* `mapping(mapper, downstream)`
  → groupingBy, joining과 함께 자주 조합됨.

---

## 3. 활용 팁

1. Collectors는 정적 임포트해서 사용하면 가독성이 좋아진다.

```
import static java.util.stream.Collectors.*;
```

2. 복잡한 조건 분기 + Map 조작 로직이 보이면
   `toMap`, `groupingBy`, `mapping`, `joining` 패턴으로 단순화할 수 있는지 확인한다.

3. Collector 조합이 너무 복잡해지면
   "차라리 명시적 for문이 낫다"고 말한다.
   선언형 코드가 항상 최선은 아니다.

4. Map을 만들 때 키 중복 발생 여부를 반드시 고려한다.
   문제가 될 여지가 1%라도 있으면 mergeFunction을 제공해야 한다.