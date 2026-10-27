# 1회용 코드 SSO와 refresh token 로테이션

> 분류: 인증·보안 · 최근 갱신: 2026-10-27

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
