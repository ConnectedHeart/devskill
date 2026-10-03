# Keycloak audience(aud)와 azp 검증

> 분류: 인증·보안 · 최근 갱신: 2026-09-30

## 한 줄 요약
`aud`는 "이 토큰이 누구를 위해 발급됐는가", `azp`는 "어느 client가 발급받았는가"이고, Keycloak은 값을 넣기만 하며 검증은 토큰을 받는 API 서버가 한다.

## 내용
### 용어
- **audience (`aud`)**: 토큰을 받아야 할 대상(API). 목록일 수 있다.
- **authorized party (`azp`)**: 토큰을 발급받은 client. "허용 client 목록" 검증은 이 값을 본다.

### 서명·issuer·만료만 검증하면 부족한 이유
JWKS로 서명, issuer, 만료만 확인하면 같은 realm의 **다른 client용으로 발급된 토큰**도 통과한다. API마다 자기 audience가 들어 있는지 확인해야 다른 서비스용 토큰의 재사용을 막을 수 있다.

### Keycloak에서 aud 넣기
- 기본 `aud`는 보통 `account`라서 검증에 쓸 수 없다.
- 토큰을 발급하는 client → Client scopes → `<client>-dedicated` → Add mapper → By configuration → **Audience** → Included Client Audience에 API 식별자 지정, "Add to access token" ON.
- 사용자에게 API client의 client role을 부여하면 그 client가 `aud`에 자동으로 들어가기도 한다.

### API 서버에서 검증
- .NET: `JwtBearerOptions.Audience` 또는 `ValidAudiences`
- Java(nimbus-jose-jwt): `DefaultJWTClaimsVerifier`에 허용 audience 집합을 넘긴다. 하나라도 일치하면 통과.

### 적용 순서
1. Keycloak에 Audience 매퍼 추가
2. 새로 발급된 토큰의 `aud`에 값이 들어왔는지 확인
3. API 서버의 audience 검증 활성화

매퍼 추가는 `aud` 목록에 값이 하나 늘어날 뿐이라 기존 사용처에 대체로 영향이 없다. API용 client를 만들기만 해서는 토큰이 바뀌지 않는다.

## 예시
```java
// 설정값이 있을 때만 audience를 검증
Set<String> accepted = audiences.isEmpty() ? null : audiences;
JWTClaimsSetVerifier<SecurityContext> verifier =
    new DefaultJWTClaimsVerifier<>(accepted, exactMatchClaims, requiredClaims, null);
```

## 주의할 점
- 토큰에 해당 audience가 없는 상태에서 검증을 먼저 켜면 모든 호출이 401이 된다. 순서를 지킨다.
- 검증 실패 사유는 로그로 남기되 토큰 값은 기록하지 않는다.
- 설정값이 비어 있으면 검증하지 않게 만들어 두면 단계적으로 적용하기 쉽다.

## 참고
- Keycloak 문서: Audience mapper, Client scopes
- RFC 7519 (JWT `aud`), OpenID Connect Core (`azp`)
