# CORS 예외 탐구

## CORS란?
브라우저가 다른 출처(origin)과 통신을 할 때
서버가 허용한다고 명시적으로 말하지 않으면 브라우저가 응답을 막는 보안 규칙.
중요한건 **서버가 아니라 브라우저가 응답을 JS에게 못주게 차단하는 것**이란 것

---
## origin(출처)란?
* schem: `http` / `https`
* host: `example.com`
* port: `:80`, `:8080`
이 모두가 **완전히 동일**해야 같은 출처로 본다. 

---
## 개발할 때 마주치는 CORS 상황
### 브라우저가 다른 도메인(origin)으로 "요청" 보낼때
* 브라우저 JS(fetch/XHR)가 다른 origin으로 요청하면 CORS 규칙 적용(일부 X)
* 서버에 전송까지는 허용, **응답을 JS가 읽을 때** CORS 헤더가 없으면 브라우저가 차단

### 다른 도메인 서버가 제공해주는 JS SDK 내부에서 발생한 통신에서 CORS 발생
* 이유: SDK는 현 페이지의 도메인을 origin으로 통신하기에 다른 도메인으로 통신 불가
* 해결:
    * form 형태로 전송함(CORS 검사 X)
    * popup / redirect : 페이지 이동은 브라우저가 보안상 허용
    * 결과는 callback / redirect / postMessage 로 전달

### CORS 정책처리해도 Preflight 때문에 CORS 발생
* 이유: 요청 전에 Preflight가 날아와서 Options도 정책에 추가해주어야함.

#### +) Simple Request(preflight X)
* method가 `GET`/`POST`/`HEAD` 중 하나
* 커스텀 헤더 안씀
* `Content-Type`이 아래 중 하나
    * `application/x-www-form-urlencoded`
    * `multipart/form-data`
    * `text/plain`

#### +) Preflight 발생
* `PUT`, `DELETE`, `PATCH` 등
* `Content-Type`: `application/json`
* `Authorization: Bearer ...`
* `fetch`에 `credentials: 'include'`

---
## Access-Controler-* 헤더
### `Access-Control-Allow-Origin`
* 어떤 origin의 js가 이 응답을 읽어도 되는지
### `Access-Control-Allow-Credentials`
* 쿠키/인증정보 포함 요청을 허용할지(true/없음)
* 중요: credentials를 쓰면 `Allow-Origin`은 `*`가 될 수 없음 → 반드시 구체 origin

---
## Spring의 CORS 설정 동작 방식
```java
<mvc:cors>
  <mvc:mapping path="/**"
    allowed-origin-patterns="*"
    allowed-methods="POST,GET"
    allow-credentials="true"/>
</mvc:cors>
```
* 요청의 origin을 매칭해서 응답에 `Access-Control-Allow-Origin: <요청 Origin>`으로 "반사"해주는 방식으로 동작
* 응답이 `*` 가 아니라 구체 Origin이 되어 Credentials=true여도 브라우저가 수용 가능

---
## CORS 왜 하는데? (공격자 입장)
### 전제
* 사용자가 `bank.com`에 로그인 되어있음
* 세션 쿠키는 브라우저에 저장됨
### 공격
1. 사용자가 `evil.com` 방문
2. `evil.com`의 JS 실행
3. JS가 브라우저에게 요청 지시: `fetch("http_://bank.com/account")
4. 브라우저: 같은 브라우저라 쿠키 포함해서 요청
5. bank.com 응답: 계좌정보
6. JS가 응답 읽어서 `evil.com`의 서버로 데이터 전송

> 다른 사이트의 응답을 읽지 못하도록 제한하는 CORS 정책 필요

---
## 왜 하필 요청? 왜 서버가 아닌 브라우저?
* 웹은 처음부터 사이트 접속하기 위함인데, 다른 도메인이니까 요청을 금지하면 웹 생태계가 붕괴
* 서버는 "요청의 의도"를 알 수 없음