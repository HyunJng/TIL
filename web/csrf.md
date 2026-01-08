# CSRF 예외 탐구
## CSRF 란?
사용자의 권한(쿠키)를 이용해 서버 상태를 몰래 변경하는 "공격"

---
## CORS 로 왜 못막아?
* 응답을 읽을 필요가 없음
* 요청만 성공하면 공격 성공

---
## CSRF 공격 어떻게 하는데?
### 전제
* 사용자는 `bank.com`에 로그인 됨
* 세션 쿠키 보유
### 공격
1. 사용자가 `evil.com` 방문 → 페이지에 `bank.com`으로 보내는 요청 존재(JS 요청이 아닌게 포인트)
    ```html
   <form action="https://bank.com/transfer" method="POST">
      <input name="to" value="attacker">
      <input name="amount" value="1000000">
    </form>
    
    <script>
      document.forms[0].submit();
    </script>
    ```
2. 브라우저는 자동으로 쿠키 포함
3. 서버는 "정상 사용자 요청"으로 인식
4. 돈 이체 완료

---
## CSRF 방어 방법 : SameSite Cookie
이 쿠키를 "다른 사이트"요청에도 실어 보낼 것인가?"를 정하는 쿠키 옵션
```http
Set-Cookie: SESSION=abc123;  SameSite=Strict
```
### SameSite 옵션 3종류
#### `SameSite=Strict`
* 완전 차단.
* 다른 사이트에서 시작된 요청에는 쿠키 절대 안보냄
* 보안은 좋지만, UX는 불편(외부 링크 타고 오면 로그인 풀림 등) 
#### `SameSite=Lax`(요즘 기본)
* 위험한 요청만 차단
* 허용:
    * 주소창 직접 입력
    * `<a href>` GET 이동
* 차단
    * `<form POST>`
    * fetch/XHR
#### `SameSite=None`
* 무조건 쿠키 전송
 