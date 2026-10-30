# 1회용 코드 SSO와 refresh token 로테이션

> 분류: 인증·보안 · 최근 갱신: 2026-10-28

## 한 줄 요약
다른 시스템을 로그인된 상태로 팝업에 띄울 때는 토큰을 브라우저로 내보내지 않고 서버끼리 1회용 코드를 주고받으며, refresh token을 요청마다 갱신하면 동시 요청에서 실패할 수 있다.

## 내용
### 1회용 코드(ticket) 방식 SSO 흐름
1. 사용자가 A 시스템에서 "B 열기"를 누른다.
2. A 서버가 보관 중인 refresh token으로 인증 서버에서 access token을 새로 받는다.
3. A 서버가 `Authorization: Bearer <토큰>`으로 B 서버의 티켓 발급 API를 호출해 수명이 짧은(예: 30초) 1회용 코드를 받는다.
4. 브라우저는 `B/sso?code=<코드>` 주소로 팝업을 연다. B는 코드를 확인하고 자기 세션을 만든다.

- access token이 자바스크립트나 주소창에 나오지 않는다. 주소에 실리는 것은 금방 만료되고 한 번만 쓸 수 있는 코드뿐이다.
- 실패하면 팝업을 열지 않고 B가 준 오류 메시지를 보여준다. 빈 팝업을 먼저 열어 두는 방식은 쓰지 않는다.

### 오류가 어느 쪽 문제인지 구분하기
| 증상 | 어디서 실패했나 |
|---|---|
| A에서 "인증 정보 없음" | A에 refresh token이 없거나 재발급 실패(만료, 연결 실패, 로테이션 충돌) |
| B가 "SSO 미설정"(503 등) | A → B 호출은 성공. B의 인증 서버 연동 설정 누락 |
| B가 "토큰 교환 실패"(502 등) | A → B 호출은 성공. B가 받은 토큰을 인증 서버에서 교환하다 실패 |

토큰 교환 실패의 흔한 원인은 realm/client 불일치, Token Exchange 권한 미설정, B의 client 인증 정보 오류, B → 인증 서버 연결 문제다. 이때 A의 코드를 고칠 것은 없고, B 로그에 찍힌 인증 서버의 실제 오류 응답(`invalid_client`, `access_denied` 등)을 봐야 한다.

### 티켓 발급 때 넘기는 자격증명: 기존 세션 대신 전용 세션
A가 자체 세션(쿠키 값)으로 인증하는 구조라면, 티켓 발급 때 B에 넘길 자격증명을 "사용자의 브라우저 로그인 세션 ID를 B용 모양으로 감싼 값"으로 만들기 쉽다. 이렇게 하면 문제가 생긴다.
- **위조와 구별할 수 없다.** 감싸는 규칙이 소스에 있고 세션 ID는 브라우저 개발자도구에서 보이므로, 일반 사용자도 자기 쿠키를 같은 모양으로 감싸 "B 관리자 전용 API"를 부를 수 있다. 서버가 이를 막으려고 "B가 등록한 세션만 인정" 같은 검사를 넣으면, 이번에는 정상 SSO 토큰까지 같은 이유로 거부된다(403).
- **로그아웃이 번진다.** B가 받은 토큰으로 로그아웃 API를 부르면 토큰이 가리키는 세션, 즉 사용자의 A 화면 세션이 지워져 A에서도 로그아웃된다.
- **수명이 묶인다.** A에서 로그아웃하거나 세션이 만료되면 B가 쥔 토큰도 함께 무효가 된다.

해결은 **티켓을 발급할 때 B 전용 세션을 새로 만들어 그 세션의 토큰을 넘기는 것**이다.
- 새 세션 ID는 서버끼리만 오가고 브라우저로 내려가지 않으므로 사용자가 만들어 낼 수 없다. 세션에 "B용으로 서버가 발급함" 표시를 함께 저장하면 방어 검사를 그대로 둘 수 있다.
- 새 행을 추가만 하므로 기존 화면 세션과 쿠키는 그대로다. 중복 로그인 제한이 "로그인 기록"으로 동작한다면 이 발급이 그 기록을 건드리지 않는지 확인한다.
- B가 티켓 발급을 거절하면(권한 없음 등) 방금 만든 세션을 바로 지운다.
- B의 로그아웃은 B 세션만 지운다. B가 로그아웃 API를 부르지 않으면 세션 수명이 지날 때까지 남으므로 수명을 짧게 두거나 정리 작업 대상에 넣는다.
- 클릭할 때마다 세션 행이 하나씩 생긴다는 점을 운영에 알려 둔다.

### refresh token 로테이션과 동시 요청
- 로테이션이 켜져 있으면(Keycloak의 "Revoke Refresh Token") refresh token은 한 번 쓰면 폐기되고 새 것으로 바뀐다.
- 화면 요청마다 refresh grant를 호출하면 동시에 들어온 두 요청이 같은 refresh token을 쓰게 되고, 늦은 쪽이 `invalid_grant`로 실패한다. 요청마다 인증 서버 왕복이 붙어 응답도 느려진다.
- "기능 사용 여부 판정"과 "토큰 재발급"을 분리해서, 토큰이 실제로 필요한 화면에서만 재발급한다.

## 핵심 코드
### 1) A 서버: refresh token으로 access token 재발급 후 B의 티켓 API 호출 (Spring RestTemplate)
```java
MultiValueMap<String, String> form = new LinkedMultiValueMap<>();
form.add("grant_type", "refresh_token");
form.add("client_id", "<client>");
form.add("client_secret", clientSecret);          // 설정·환경 변수에서 읽는다
form.add("refresh_token", storedRefreshToken);
Map<?, ?> tok = rest.postForObject(
    "<서버 주소>/realms/<realm>/protocol/openid-connect/token", form, Map.class);
// 로테이션이 켜져 있으면 응답의 새 refresh_token을 반드시 저장 (이전 값은 폐기됨)
saveRefreshToken(userId, (String) tok.get("refresh_token"));

HttpHeaders h = new HttpHeaders();
h.setBearerAuth((String) tok.get("access_token"));
Map<?, ?> ticket = rest.postForObject(
    "https://b.example.com/api/sso/ticket", new HttpEntity<Void>(h), Map.class);
return ticket.get("code");                         // 브라우저에는 코드만 돌려준다
```

### 2) B 서버: 1회용 코드 발급 (수명 30초)
```java
String code = UUID.randomUUID().toString();       // SecureRandom 기반이라 추측 불가
jdbc.update("INSERT INTO sso_ticket (code, user_id, expires_at, used) VALUES (?, ?, ?, 0)",
    code, userId, new Timestamp(System.currentTimeMillis() + 30_000L));
```

### 3) B 서버: 코드 사용 — 조건부 UPDATE로 한 번만 통과
```java
Timestamp now = new Timestamp(System.currentTimeMillis());
int n = jdbc.update(
    "UPDATE sso_ticket SET used = 1 WHERE code = ? AND used = 0 AND expires_at > ?", code, now);
if (n != 1) { resp.sendError(401); return; }       // 없음·만료·재사용 모두 여기서 거부
String userId = jdbc.queryForObject(
    "SELECT user_id FROM sso_ticket WHERE code = ?", String.class, code);

HttpSession old = req.getSession(false);
if (old != null) old.invalidate();                 // 세션 고정 방지
req.getSession(true).setAttribute("loginUserId", userId);
```

### 4) B 서버: 받은 토큰을 인증 서버에서 교환 (Token Exchange, RFC 8693)
```java
MultiValueMap<String, String> form = new LinkedMultiValueMap<>();
form.add("grant_type", "urn:ietf:params:oauth:grant-type:token-exchange");
form.add("subject_token", incomingAccessToken);
form.add("subject_token_type", "urn:ietf:params:oauth:token-type:access_token");
form.add("client_id", "<B client>");
form.add("client_secret", bClientSecret);
try {
    Map<?, ?> exchanged = rest.postForObject(tokenUrl, form, Map.class);
} catch (HttpClientErrorException e) {
    // invalid_client, access_denied 등 인증 서버의 실제 오류를 남겨야 원인이 보인다
    log.warn("token exchange failed status={} body={}", e.getStatusCode().value(), e.getResponseBodyAsString());
    return ResponseEntity.status(502).body(Collections.singletonMap("message", "토큰 교환 실패"));
}
```

### 5) 브라우저: 코드를 받은 뒤에만 팝업을 연다
```javascript
fetch('/sso/b-ticket', { method: 'POST', credentials: 'same-origin' })
  .then(function (r) { return r.json(); })
  .then(function (d) {
    if (d.code) {
      window.open('https://b.example.com/sso?code=' + encodeURIComponent(d.code), 'b');
    } else {
      alert(d.message);                            // B가 준 오류 메시지 표시
    }
  });
```

### 5-1) A 서버: 티켓 발급용 전용 세션 만들기 (기존 세션을 감싸지 않음)
```java
// 바꿔야 할 곳: 세션 테이블·컬럼 이름, 토큰 암호화 방식, B 티켓 API 주소
String ssoSessionId = UUID.randomUUID().toString();                // 브라우저로 내려가지 않는 새 ID
jdbc.update("INSERT INTO user_session (session_id, user_id, issued_for, expires_at) VALUES (?, ?, 'b-console', ?)",
    ssoSessionId, loginUser.getUserId(), new Timestamp(System.currentTimeMillis() + 60 * 60_000L));
String token = tokenCipher.encrypt(ssoSessionId);                 // 이 세션만 가리키는 토큰

try {
    String code = requestTicket(token);                           // 위 1)처럼 Bearer 로 B 티켓 API 호출
    if (code == null) {                                           // B가 거절(권한 없음 등)
        jdbc.update("DELETE FROM user_session WHERE session_id = ?", ssoSessionId);
        return error("B 에서 거절함");
    }
    return ok(code);
} catch (RuntimeException e) {
    jdbc.update("DELETE FROM user_session WHERE session_id = ?", ssoSessionId);
    throw e;
}
```

### 5-2) API 서버: 관리자 전용 API는 "서버가 발급한 세션"만 인정
```java
// 토큰 모양만 보지 않고, 가리키는 세션이 B용으로 서버가 발급한 것인지 DB 로 확인
List<String> rows = jdbc.queryForList(
    "SELECT issued_for FROM user_session WHERE session_id = ? AND expires_at > ?",
    String.class, sessionId, new Timestamp(System.currentTimeMillis()));
boolean consoleSession = rows.size() == 1 && "b-console".equals(rows.get(0));   // 없거나 만료면 거부
```

### 6) 인증 서버 호출에 짧은 타임아웃
```java
SimpleClientHttpRequestFactory f = new SimpleClientHttpRequestFactory();
f.setConnectTimeout(2000);   // ms
f.setReadTimeout(5000);
RestTemplate rest = new RestTemplate(f);
```

## 주의할 점
- refresh token 쿠키가 로그인 시점에만 만들어지는 구조라면, 비밀번호 없이 들어오는 SSO 로그인이나 기능을 켜기 전에 로그인한 세션에는 쿠키가 없다. "인증 정보 없음"의 흔한 원인이다.
- 인증 서버 호출에는 짧은 타임아웃을 두고, 실패 로그 문구를 원인별로 구분해 두면 추적이 쉽다.

## 참고
- OAuth 2.0 Token Exchange (RFC 8693)
- Keycloak 문서: Refresh token rotation ("Revoke Refresh Token")
