## item 80. 스레드보다는 실행자, 테스크,스트림을 애용하라 
> * 스레드는 직접 다루는 대상이 아니다
> * **태스크(Runnable/Callable)를 정의하고**
> * **실행은 Executor에 위임하라**
> * 반환값은 즉시 값이 아니라 **Future로 모델링하라**

## 1. Item 80 핵심 메시지

* 스레드를 직접 생성하고 관리하지 말고
* **작업(Task)**과 **실행 정책(Executor)**을 분리하라
* 스레드는 구현 세부사항이지, API가 아니다

---

## 2. 스레드 중심 사고의 문제점

전통적인 방식:

```java
new Thread(() -> doSomething()).start();
```

이 방식의 문제:

* 스레드 생성 비용을 호출자가 떠안음
* 스레드 개수 조절 불가
* 재사용 불가
* 예외 처리, 종료 정책 제어 어려움
* 반환값 처리 구조가 없음

👉 **“무엇을 실행할지”와 “어떻게 실행할지”가 뒤섞여 있다**

---

## 3. Executor 모델의 핵심 분리 개념

* Runnable / Callable
  → **무엇을 할지 (Task)**
* Executor / ExecutorService
  → **어떻게 실행할지 (Policy)**
* Thread
  → **구현 세부사항**

이제 호출자는 **작업만 정의**하고,
실행 전략은 Executor에게 위임한다.

---

## 4. Runnable vs Callable — 동기/비동기와 무관하다

### 4.1 Runnable

* 반환값 없음
* 체크 예외 불가
* 단순 실행용 작업 표현

```java
Runnable task = () -> doSomething();
```

### 4.2 Callable

* 반환값 있음
* 체크 예외 가능
* “값을 계산하는 작업” 표현

```java
Callable<Integer> task = () -> 42;
```

### 4.3 핵심 오해 정리

* Runnable / Callable의 차이 ≠ 동기 / 비동기
* 차이는 **“반환값 모델링 가능 여부”**다
* 실행 시점(동기/비동기)은 Executor가 결정한다

---

## 5. 비동기 실행 + 반환값이 가능한 이유

### 5.1 Future의 역할

Callable을 Executor에 제출하면:

```java
Future<Integer> future = executor.submit(callable);
```

* 실행은 **다른 스레드에서 비동기**
* 반환값은 즉시 받지 않고
* **Future라는 “미래의 결과 핸들”을 즉시 반환**

### 5.2 동기 지점은 선택 사항

```java
Integer result = future.get(); // 여기서만 블로킹
```

* 비동기 실행은 이미 시작됨
* `get()`을 호출하는 순간만 동기 대기
* 호출하지 않으면 끝까지 비동기

👉 **비동기 실행과 결과 수신 시점은 분리된다**

---

## 6. Executor는 “개별 작업자”가 아니라 “작업 조직”이다

중요한 개념 정리:

* Thread
  → 개별 작업자
* Runnable / Callable
  → 개별 작업
* Executor / ThreadPoolExecutor
  → 작업을 받아서 작업자에게 배분하는 관리자

그래서:

```java
Executor executor = new ThreadPoolExecutor(...);
```

이 구조가 **정상적이고 권장되는 방식**이다.

Executor를 반환한다는 것은:

* “스레드 하나를 준다”가 아니라
* “작업을 맡길 수 있는 실행 시스템을 준다”는 의미다

---

## 7. 스프링 @Async와 Item 80의 연결

스프링의 `@Async`는 Item 80 사상을 그대로 구현한 사례다.

* 프록시가 메서드 호출을 가로챔
* 호출 자체를 Runnable / Callable 태스크로 변환
* Executor에 submit
* 실행 정책은 전부 Executor가 담당

즉:

* 개발자는 **작업 정의(@Async 메서드)**에만 집중
* 실행 전략(스레드 수, 큐, 종료 정책)은 설정으로 분리
