# Spring의 TheradPool 매번 헷갈리는 부분 노트
> Async ThreadPool 설정 관련은 [망규님 블로그 확인](https://mangkyu.tistory.com/425)

## 헷갈리는 Pool 설정들
### DB 커넥션 Pool
```properties
spring:
  datasource:
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
```
* 히카리CP 커넥션풀 설정

```
[Tomcat Thread]
   ↓
[Service / Repository 코드 실행]
   ↓
[HikariCP에서 Connection 하나 빌림]
   ↓
[DB 쿼리 수행]
```

### Spring기본: Tomcat Thread Pool
```properties
server:
  tomcat:
    threads:
      max: 200
```
* HTTP 요청을 처리하는 Thread Pool

### 비동기(Async) Thread Pool
```java
@Bean
public Executor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(10);
    executor.setMaxPoolSize(50);
    return executor;
}
```
* tomcat 스레드는 요청용 처리
* async용으로 만든 스레드풀은 백그라운드 처리용.

---
## 비동기마 스레드 풀 설정 방법
비동기 기능마다 스레드풀을 따로 만들면 보통 효율이 나빠진다
* 스레드 수 총합이 늘어남 → 컨텍스트 스위칭 증가, CPU 낭비
* 각 풀마다 큐/스레드/모니터링 포인트가 생김 → 운영 복잡도↑
* DB/외부 API 같은 공통 병목은 그대로인데 스레드만 늘어나서 대기만 늘어나는 경우가 많음
* 메모리도 풀마다 스레드 스택 등으로 계속 먹음

실무는
* 기본 Async Executor 1개를 만들어서 대부분의 @Async가 공유
* 정말 성격이 다른 작업만 몇 개로만 분리 (보통 2~3개 수준)

> * 같은 풀 안에서는 보통 “빨리”를 보장하기 어렵다(큐에 쌓인 순서, 경쟁 상황에 따라).
> * 풀을 분리하면 “다른 작업에게 밀려서 못 도는 상황”을 막을 수 있어서 **사실상 우선권(격리/보장)** 을 줄 수 있다.