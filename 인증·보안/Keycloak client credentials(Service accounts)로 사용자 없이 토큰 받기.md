# Keycloak client credentials(Service accounts)로 사용자 없이 토큰 받기

> 분류: 인증·보안 · 최근 갱신: 2026-11-18

## 한 줄 요약
야간 배치처럼 사용자가 비밀번호를 입력하지 않는 곳에서 Keycloak 토큰이 필요하면, DB의 비밀번호 해시나 오래된 refresh token은 쓸 수 없고 client의 "Service accounts"를 켜서 client ID·secret만으로 받는 client credentials 방식을 검토한다. 이 토큰의 주인은 사람이 아니라 `service-account-<clientId>`다.

## 내용
### 발급 방식 비교
| grant | 넘기는 것 | 쓰는 곳 |
|---|---|---|
| password | 사용자 ID + 비밀번호 원문 (+ client ID·secret) | 화면 로그인 순간 |
| refresh_token | 이전에 받은 refresh token | 만료 전 재발급. 수명(예: 8시간)이 지나면 못 씀 |
| client_credentials | client ID + secret만, 사용자 없음 | 서버 간 호출, 배치 |
| token exchange | client 인증 + 대상 사용자 | 다른 사용자 명의 토큰. 서버에서 기능·권한 설정 필요 |

### DB 비밀번호로는 안 되는 이유
- 앱 DB에 저장된 값은 보통 SHA-256 같은 단방향 해시(사용자 ID 등을 솔트로)다. 원문으로 되돌릴 수 없고, Keycloak은 해시를 비밀번호로 받지 않는다.
- 같은 이유로 SSO로 로그인해 비밀번호를 입력하지 않은 사용자에게는 password grant 토큰을 만들 수 없다.

### client와 user
- Keycloak 계정은 사람(user: ID+비밀번호)과 프로그램(client: client ID+secret) 두 종류다.
- 앱이 이미 client secret을 갖고 있어도, 평소에는 "등록된 앱"이라는 확인용으로만 쓰고 토큰은 사용자 명의로 받는다.
- client 설정의 "Service accounts roles"(Client authentication 켜짐 필요)를 켜면 client 자신의 명의로 토큰을 받을 수 있다.

### 받은 뒤 걸리는 점
- 토큰의 `sub`·`preferred_username`이 `service-account-<clientId>`다. 받는 API 서버가 이 계정을 관리자 등 필요한 권한으로 인정해야 한다(역할 매핑 또는 상대 시스템 등록).
- secret의 무게가 커진다. 옵션을 켜기 전에는 secret만으로 아무 사용자 데이터도 못 봤지만, 켠 뒤에는 secret을 가진 사람이 서비스 계정 권한(예: 전 직원 데이터)의 토큰을 받을 수 있다. 설정 파일 접근 권한이 곧 그 권한이 된다.
- 사이트(realm)마다 관리자가 옵션을 켜야 하므로 배포 절차에 넣는다.

## 핵심 코드
### 1) 토큰 요청 (HTTP)
```http
POST https://<keycloak>/realms/<realm>/protocol/openid-connect/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&client_id=<client-id>&client_secret=<client-secret>
```

### 2) 배치에서 받기 (Java, Spring RestTemplate)
```java
MultiValueMap<String, String> form = new LinkedMultiValueMap<>();
form.add("grant_type", "client_credentials");
form.add("client_id", clientId);          // 설정에서 읽음
form.add("client_secret", clientSecret);  // 로그에 남기지 않음
HttpHeaders h = new HttpHeaders();
h.setContentType(MediaType.APPLICATION_FORM_URLENCODED);
Map<?, ?> res = restTemplate.postForObject(tokenUrl, new HttpEntity<>(form, h), Map.class);
String accessToken = (String) res.get("access_token");   // refresh token은 보통 오지 않음, 매번 새로 받음
```

### 3) 응답에 사용자 ID 접미사가 붙어 올 때 안전하게 떼기
```java
// "user@company" 형식이면 바깥 companyId와 같은 접미사일 때만 뗀다(이메일형 ID 보호)
String suffix = "@" + companyId;
String userId = rawUserId.endsWith(suffix)
        ? rawUserId.substring(0, rawUserId.length() - suffix.length())
        : rawUserId;
```

## 주의할 점
- client credentials 응답에는 보통 refresh token이 없다. 만료되면 같은 요청으로 다시 받는다.
- 비밀번호 원문을 설정에 넣어 password grant로 받는 방법도 있지만, 평문 비밀값 보관과 비밀번호 변경 시 배치 중단 문제가 있다.
- 토큰·secret은 로그와 화면에 출력하지 않는다.

## 참고
- Keycloak Server Administration Guide: Service accounts, OAuth 2.0 Client Credentials Grant
- RFC 6749 4.4 Client Credentials Grant, RFC 8693 Token Exchange
