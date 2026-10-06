# Keycloak 토큰 클레임(jti·sid·sub)과 로그아웃 범위

> 분류: 인증·보안 · 최근 갱신: 2026-10-06

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

## 예시
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

검증 실험 설계: 같은 사용자로 두 번 로그인(세션 A, B) → A에서 두 번 재발급 → A만 로그아웃 → 모든 토큰을 introspect. A의 토큰만 전부 `active=false`이고 B가 살아 있으면 세션 단위임이 확인된다.

## 주의할 점
- 토큰 수명, refresh token 폐기 여부는 realm 설정이라 환경(개발/운영)마다 다를 수 있다. 한 환경의 측정값을 다른 환경에 그대로 적용하지 않는다.
- 실험 스크립트는 토큰 원문을 출력하지 말고 클레임과 비교 결과만 출력한다. 유효한 토큰이 터미널 기록에 남는다.
- 로그아웃을 즉시 반영해야 하는 API라면 introspection을 쓰거나 access token 수명을 짧게 잡아야 한다.

## 참고
- RFC 7519 (JSON Web Token) — `jti`, `sub`, `exp`, `iat`
- RFC 7662 (OAuth 2.0 Token Introspection)
- Keycloak Server Administration Guide — Sessions, Tokens
