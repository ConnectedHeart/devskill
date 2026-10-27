# Keycloak 토큰 클레임(jti·sid·sub)과 로그아웃 범위

> 분류: 인증·보안 · 최근 갱신: 2026-10-27

## 한 줄 요약
재발급할 때마다 access token은 새로 만들어지지만 같은 로그인 세션(`sid`)에 속하고, 로그아웃은 "사용자"가 아니라 그 "세션"의 토큰만 무효로 만든다.

## 내용
**세 가지 식별 클레임**

| 클레임 | 무엇을 구분하나 | 바뀌는 시점 |
|---|---|---|
| `sub` | 사용자 | 바뀌지 않음 |
| `sid` | 로그인 세션 | 사용자가 새로 인증할 때마다 |
| `jti` | 토큰 한 장 (JWT ID, RFC 7519) | 토큰을 발급할 때마다 |

- 아이디·비밀번호(password grant)나 로그인 화면으로 새로 인증하면 서버에 세션이 하나 생기고, 그 뒤 받는 토큰은 모두 그 번호를 `sid`로 달고 나온다.
- refresh token으로 재발급(refresh grant)하면 세션은 새로 생기지 않는다. `jti`·`iat`·`exp`만 달라지고 `sid`·`sub`·`azp`·`aud`는 그대로다. 서명 대상이 달라지므로 토큰 문자열 전체가 바뀐다.
- 같은 사용자가 다른 브라우저나 기기에서 로그인하면 `sid`가 다른 별도 세션이다.

**재발급과 이전 토큰**
- 새 access token을 받아도 이전 access token은 폐기되지 않고 자기 `exp`까지 유효하다.
- refresh token은 재발급 때 새 값이 나오지만, 이전 값을 폐기할지는 realm 설정(Revoke Refresh Token, 재사용 횟수)에 달려 있다. 폐기하지 않는 설정이면 여러 창이 동시에 재발급해도 충돌하지 않는 대신, 유출된 refresh token이 교체로 무력화되지 않는다.
- refresh token 수명은 재발급할 때마다 다시 늘어난다(세션 유휴 시간 기준).

**로그아웃 범위**
- 로그아웃 엔드포인트는 세션 단위로 동작한다. 그 세션에서 나온 access token(재발급 전의 오래된 것 포함)과 refresh token이 모두 무효가 되고, 같은 사용자의 다른 세션은 영향을 받지 않는다.
- "무효"는 인증 서버에 물었을 때의 답이다. 토큰의 서명과 `exp`는 로그아웃 후에도 그대로라서, 인증 서버에 묻지 않고 서명·만료만 자체 검증하는 API 서버는 만료 시각까지 그 토큰을 통과시킨다. access token 수명이 길수록 이 틈이 커진다.

**애플리케이션 세션과는 별개**
- WAS의 세션 쿠키(`JSESSIONID`)와 `sid`는 만드는 서버도 저장 위치도 다르고 서로 연결돼 있지 않다. 애플리케이션이 자기 로그아웃 처리에서 인증 서버의 로그아웃을 따로 호출하지 않으면, 화면에서는 로그아웃됐어도 인증 서버 세션과 토큰은 수명까지 살아 있다.

**애플리케이션 로그아웃에 인증 서버 로그아웃 붙이기**
- Keycloak은 OpenID Connect 구현체라 로그아웃(`…/protocol/openid-connect/logout`)과 토큰 폐기(`…/revoke`, RFC 7009)가 표준으로 들어 있다. 인증 서버 쪽에 새로 만들 것은 없고, 애플리케이션 로그아웃 처리에서 저장해 둔 refresh token으로 로그아웃 API를 한 번 부르면 된다. 필요한 값(client_id, client_secret, refresh_token)은 토큰을 발급받을 때 이미 갖고 있다.
- 호출 결과는 세 가지로 나눠 처리한다.

  | 응답 | 의미 | 처리 |
  |---|---|---|
  | 2xx | 세션 종료 | 디버그 로그 |
  | 400 + `invalid_grant` | refresh token 만료나 세션이 이미 끝남 | 이미 로그아웃된 것으로 보고 디버그 로그 |
  | 그 밖(401 client 인증 실패, 5xx, 시간 초과, 연결 실패) | 설정 오류나 장애 | 상태 코드만 경고 로그 |

- 어느 경우든 애플리케이션 로그아웃과 자체 토큰 저장소 정리는 계속 진행한다. 인증 서버 호출은 짧은 시간 제한을 두고 별도 스레드에 넘겨 응답을 기다리지 않으면, 인증 서버가 느려도 로그아웃이 늦어지지 않는다.
- 같은 브라우저에서 다시 로그인해 저장 토큰을 갈아 끼울 때도 이전 세션을 같은 방식으로 끝내야 인증 서버에 세션이 쌓이지 않는다.
- 이것으로 풀리는 것은 "로그인마다 세션이 쌓이고 재발급이 계속 가능한 문제"까지다. 이미 다른 서버에 넘긴 access token은 위에서 말한 대로 만료까지 쓰일 수 있다.

## 핵심 코드
### 1) 요청 형태: 유효성 확인과 세션 로그아웃
토큰 유효성을 인증 서버에 직접 묻는 두 가지 방법:

```
# 1) introspection: active=true/false
POST <서버 주소>/realms/<realm>/protocol/openid-connect/token/introspect
  client_id=<client>&client_secret=<secret>&token=<access_token>

# 2) userinfo: 200이면 통과, 401이면 거부
GET <서버 주소>/realms/<realm>/protocol/openid-connect/userinfo
  Authorization: Bearer <access_token>

# 세션 로그아웃
POST <서버 주소>/realms/<realm>/protocol/openid-connect/logout
  client_id=<client>&client_secret=<secret>&refresh_token=<refresh_token>
```

### 2) curl로 실행 (토큰 원문 대신 클레임만 출력)
```bash
# introspection: client 인증은 Basic으로, 결과에서 판정에 필요한 값만
curl -s -u "<client>:$CLIENT_SECRET" -d "token=$ACCESS_TOKEN" \
  "<서버 주소>/realms/<realm>/protocol/openid-connect/token/introspect" | jq '{active, sub, sid, jti}'

# 세션 로그아웃: 성공하면 204
curl -s -o /dev/null -w "%{http_code}\n" -d "client_id=<client>" -d "client_secret=$CLIENT_SECRET" \
  -d "refresh_token=$REFRESH_TOKEN" "<서버 주소>/realms/<realm>/protocol/openid-connect/logout"
```

### 3) 애플리케이션 로그아웃에 인증 서버 로그아웃 붙이기 (비동기, 짧은 타임아웃)
```java
SimpleClientHttpRequestFactory f = new SimpleClientHttpRequestFactory();
f.setConnectTimeout(2000);
f.setReadTimeout(3000);
RestTemplate rest = new RestTemplate(f);
ExecutorService logoutPool = Executors.newFixedThreadPool(2);

public void logoutAtIdp(final String refreshToken) {
    logoutPool.submit(() -> {                       // 응답을 기다리지 않는다
        MultiValueMap<String, String> form = new LinkedMultiValueMap<>();
        form.add("client_id", clientId);
        form.add("client_secret", clientSecret);
        form.add("refresh_token", refreshToken);
        try {
            rest.postForEntity(logoutUrl, form, String.class);
            log.debug("idp logout ok");
        } catch (HttpClientErrorException e) {
            if (e.getStatusCode().value() == 400 && e.getResponseBodyAsString().contains("invalid_grant")) {
                log.debug("idp session already ended");
            } else {
                log.warn("idp logout failed status={}", e.getStatusCode().value());
            }
        } catch (RestClientException e) {           // 5xx, 시간 초과, 연결 실패
            log.warn("idp logout failed: {}", e.getClass().getSimpleName());
        }
    });
}
// 호출하는 쪽: logoutAtIdp(saved) 후 바로 자체 토큰 저장소 정리와 세션 무효화를 계속한다
```

### 4) 실험용: 토큰에서 식별 클레임만 꺼내 출력
```java
static String ids(String jwt) throws IOException {
    JsonNode p = new ObjectMapper().readTree(Base64.getUrlDecoder().decode(jwt.split("\\.")[1]));
    return "sub=" + p.path("sub").asText() + " sid=" + p.path("sid").asText()
         + " jti=" + p.path("jti").asText() + " exp=" + p.path("exp").asLong();
}
```

검증 실험 설계: 같은 사용자로 두 번 로그인(세션 A, B) → A에서 두 번 재발급 → A만 로그아웃 → 모든 토큰을 introspect. A의 토큰만 전부 `active=false`이고 B가 살아 있으면 세션 단위임이 확인된다.

## 주의할 점
- 토큰 수명, refresh token 폐기 여부는 realm 설정이라 환경(개발/운영)마다 다를 수 있다. 한 환경의 측정값을 다른 환경에 그대로 적용하지 않는다.
- 실험 스크립트는 토큰 원문을 출력하지 말고 클레임과 비교 결과만 출력한다. 유효한 토큰이 터미널 기록에 남는다.
- 로그아웃을 즉시 반영해야 하는 API라면 introspection을 쓰거나 access token 수명을 짧게 잡아야 한다.

## 참고
- RFC 7519 (JSON Web Token) — `jti`, `sub`, `exp`, `iat`
- RFC 7662 (OAuth 2.0 Token Introspection)
- RFC 7009 (OAuth 2.0 Token Revocation)
- OpenID Connect RP-Initiated Logout
- Keycloak Server Administration Guide — Sessions, Tokens
