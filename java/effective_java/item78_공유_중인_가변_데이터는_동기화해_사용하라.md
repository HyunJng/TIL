## item 78. 공유 중인 가변 데이터는 동기화해 사용하라
> * 공유 가변 데이터는 동기화가 반드시 필요하다.
> * 동기화는 배타성(쓰기)만을 위한 것이 아니라 가시성(읽기)과 순서를 보장하기 위한 메커니즘이며,
    >   이 효과는 쓰기와 읽기 양쪽이 모두 동기화에 참여해야만 성립한다.

---

## 1. 공유 가변 변수를 동기화하지 않으면 발생하는 문제

```java
private static boolean stopRequested;

public static void main(String[] args) throws InterruptedException {
    Thread worker = new Thread(() -> {
        while (!stopRequested) {
            // do something
        }
    });
    worker.start();

    Thread.sleep(1000);
    stopRequested = true;
}
```

직관적으로는:

* main 스레드가 `stopRequested = true`로 변경
* worker 스레드가 이를 보고 while 루프 종료

라고 생각하기 쉽다.

하지만 **동기화가 없으면 worker 스레드는 영원히 종료되지 않을 수 있다.**

> 메인 스레드가 종료돼도 프로그램이 끝나지 않는 이유
> * `new Thread()`로 생성한 스레드는 기본적으로 **non-daemon 스레드**
> * JVM은 **non-daemon 스레드가 하나라도 살아 있으면 종료되지 않음**
> * 따라서 main 스레드가 끝나도 worker 스레드가 무한 루프면 프로그램은 계속 실행됨

---

## 2. 각 스레드는 자기만의 작업 메모리를 사용한다

### 2.1 힙 / 스택과의 관계

* 힙
    * 객체, 클래스 필드
    * 모든 스레드가 공유
* 스택
    * 지역 변수, 메서드 호출 프레임
    * 스레드별로 독립

`stopRequested`는 **힙에 있는 공유 변수**다.

하지만 문제가 발생하는 이유는 **위치(힙/스택)가 아니라 접근 방식** 때문이다.

---

### 2.2 자바 메모리 모델(JMM)의 작업 메모리

JMM에서는 다음과 같이 추상화한다.

* 메인 메모리

    * 공유 변수의 “정본”
* 작업 메모리 (스레드별)

    * CPU 캐시
    * 레지스터
    * JIT이 만든 임시 값 등

스레드는:

* 메인 메모리에서 값을 읽어 작업 메모리에 복사
* 이후에는 작업 메모리 값만 사용할 수 있음
* 언제 다시 메인 메모리를 읽을지는 보장되지 않음

---

## 3. stopRequested가 영원히 false로 보일 수 있는 이유

### 3.1 가시성 문제

* worker 스레드

    * `stopRequested = false`를 한 번 읽어 작업 메모리에 저장
* main 스레드

    * `stopRequested = true`로 변경
    * 메인 메모리에 반영
* worker 스레드

    * 다시 메인 메모리를 읽지 않음
    * 계속 false만 사용

결과:

* 힙에는 true
* worker의 관점에서는 false

---

## 4. 쓰기만 동기화해서는 부족하다

많이 하는 실수:

```java
synchronized void requestStop() {
    stopRequested = true;
}

void run() { // 동기화 없음
    while (!stopRequested) {
        work();
    }
}
```

이 경우:

* writer는 값을 메인 메모리에 flush
* reader는 락을 획득하지 않으므로 reload 보장 없음

즉,

* 쓰기와 읽기 사이에 **happens-before 관계가 성립하지 않는다**
* reader는 여전히 옛값을 볼 수 있다

---

## 5. synchronized의 진짜 역할

`synchronized`는 단순히 “동시에 못 들어오게 한다”가 아니다.

* 락 획득 시

    * 작업 메모리 무효화
    * 메인 메모리에서 다시 읽음
* 락 해제 시

    * 변경 내용을 **메인 메모리에 반영**

따라서:

* **쓰기 쪽과 읽기 쪽 모두 동기화에 참여해야**
* happens-before가 성립한다

---

## 6. 해결 방법

### 6.1 읽기와 쓰기 모두 synchronized

```java
synchronized boolean isStopRequested() {
    return stopRequested;
}

synchronized void requestStop() {
    stopRequested = true;
}
```

---

### 6.2 volatile 사용

```java
private static volatile boolean stopRequested;
```

volatile이 보장하는 것:

* 가시성 보장
* 재주문 방지
* 매 읽기마다 메인 메모리 기준 값 사용

---

## 7. JPA 락과의 개념적 유사성

### JPA 비관 락 예시

* `findForUpdate()` → DB 락에 참여
* `findById()` → 락에 참여하지 않음

락에 참여하지 않는 접근 경로가 하나라도 있으면:

* 락의 의미가 무너짐
* 데이터 정합성 보장 불가