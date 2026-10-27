# JWT 구조와 서명 검증 조건

> 분류: 인증·보안 · 최근 갱신: 2026-10-27

## 한 줄 요약
흔히 보는 JWT(access token)는 암호화된 것이 아니라 서명만 된 것이라 누구나 내용을 읽을 수 있고, 서버는 "복호화"가 아니라 서명·발급자·만료·대상(audience)을 검증한다.

## 내용
### 구조
- JWT는 `헤더.페이로드.서명` 세 부분을 점으로 이은 문자열이다.
- 헤더와 페이로드는 JSON을 Base64URL로 인코딩한 것뿐이다. 키 없이 디코딩하면 그대로 읽힌다. 그래서 토큰에 비밀 값을 넣으면 안 되고, 토큰을 채팅·로그·이슈에 붙여 넣는 것도 내용을 공개하는 것과 같다.
- 서명은 헤더와 페이로드를 발급 서버의 개인키(RS256 등)로 서명한 값이다. 내용을 한 글자라도 바꾸면 서명이 맞지 않는다.
- 내용까지 숨기는 형식은 JWE(암호화된 JWT)로 따로 있고, 점으로 나뉜 부분이 5개다.

### 헤더·페이로드에서 자주 보는 값
| 값 | 의미 |
|---|---|
| `alg` | 서명 알고리즘 (예: RS256) |
| `kid` | 서명에 쓴 키의 ID. 검증 서버는 이 ID로 공개키를 고른다 |
| `iss` | 발급자. Keycloak이면 `<서버 주소>/realms/<realm>` |
| `aud` | 이 토큰을 받아도 되는 대상. 여러 개일 수 있다 |
| `azp` | 토큰을 요청한 클라이언트 |
| `iat` / `exp` | 발급 시각 / 만료 시각 (epoch 초) |
| `preferred_username` 등 | 사용자 식별 정보 |

### API 서버가 검증하는 것
1. **서명** — 발급 서버의 공개키 목록(JWKS 주소)에서 `kid`에 맞는 키를 받아 서명을 확인한다. API 서버가 발급 서버에 네트워크로 접속할 수 있어야 한다(보통 받아 온 키는 캐시한다).
2. **발급자(`iss`)** — 설정해 둔 발급자 문자열과 정확히 같아야 한다. 프로토콜(http/https), 포트, 끝의 슬래시, 내부 주소와 외부 주소 차이만으로도 불일치가 된다.
3. **만료(`exp`)** — 서버 시계 차이를 감안한 허용 오차(clock skew, 보통 수십 초)를 둔다. access token 수명이 1분이면 허용 오차를 더해도 발급 후 2분 안팎까지만 통과한다.
4. **대상(`aud`)** — 검증할 audience를 설정했다면 그 값이 토큰의 `aud`에 들어 있어야 한다. 설정하지 않으면 검사하지 않는 구현이 많은데, 그러면 같은 발급 서버의 다른 서비스용 토큰도 통과한다.

검증에 성공하면 페이로드의 클레임을 그대로 꺼내 쓴다. 따로 복호화 단계는 없다.

### 받은 토큰이 401일 때 순서대로 볼 것
- 서버가 JWT 검증 방식으로 설정돼 있는가(다른 인증 방식으로 설정돼 있으면 JWT를 아예 해석하지 않는다)
- 토큰을 디코딩해 `exp`가 지났는지 — 수명이 짧은 토큰은 복사해 오는 사이에 만료된다
- `iss`가 서버 설정과 글자 그대로 같은지
- `aud`에 서버가 요구하는 값이 있는지
- 서버에서 JWKS 주소로 접속이 되는지

## 핵심 코드
### 1) 의존성 (Maven)
```xml
<dependency>
  <groupId>com.nimbusds</groupId>
  <artifactId>nimbus-jose-jwt</artifactId>
  <version>${nimbus.version}</version>  <!-- 사용 중인 Java 버전을 지원하는 버전으로 -->
</dependency>
```

### 2) API 서버: 서명·iss·exp·aud 검증 (알고리즘 고정)
```java
DefaultJWTProcessor<SecurityContext> p = new DefaultJWTProcessor<>();
JWKSource<SecurityContext> jwks = new RemoteJWKSet<>(
    new URL("<서버 주소>/realms/<realm>/protocol/openid-connect/certs"));   // JWKS 주소
// RS256만 허용 → alg=none·HS256 등은 거부된다
p.setJWSKeySelector(new JWSVerificationKeySelector<>(JWSAlgorithm.RS256, jwks));

DefaultJWTClaimsVerifier<SecurityContext> v = new DefaultJWTClaimsVerifier<>(
    Collections.singleton("<api-audience>"),                                // aud
    new JWTClaimsSet.Builder().issuer("<서버 주소>/realms/<realm>").build(), // iss 정확히 일치
    new HashSet<>(Arrays.asList("sub", "iat", "exp")),                      // 필수 클레임
    null);
v.setMaxClockSkew(30);                                                      // 초
p.setJWTClaimsSetVerifier(v);

try {
    JWTClaimsSet claims = p.process(token, null);   // 성공하면 클레임을 그대로 쓴다
    String user = claims.getStringClaim("preferred_username");
} catch (BadJOSEException | JOSEException | ParseException e) {
    log.info("jwt rejected: {}", e.getMessage());  // 토큰 값은 남기지 않는다
}
```

### 3) iss·JWKS 주소 확인 (API 서버에서 실행)
```bash
# issuer 값을 글자 그대로 확인
curl -s <서버 주소>/realms/<realm>/.well-known/openid-configuration | jq -r '.issuer, .jwks_uri'
# API 서버에서 JWKS 주소에 접속되는지, kid 목록
curl -s <서버 주소>/realms/<realm>/protocol/openid-connect/certs | jq -r '.keys[].kid'
```

### 4) 토큰 내용 읽기
```bash
# 페이로드만 읽기 (Base64URL → JSON). 패딩이 없어 오류가 나면 '='를 덧붙인다
echo '<토큰>' | cut -d. -f2 | tr '_-' '/+' | base64 -d

# epoch 초를 시각으로
date -d @1700000000
```
```powershell
$p = '<토큰>'.Split('.')[1].Replace('-','+').Replace('_','/')
$p += '=' * ((4 - $p.Length % 4) % 4)
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($p))
```

## 주의할 점
- 디코딩해서 읽을 수 있다는 것과 믿을 수 있다는 것은 다르다. 서명 검증 없이 페이로드 값을 신뢰하면 안 된다.
- 실제 사용자의 토큰을 온라인 디코딩 사이트에 붙여 넣지 않는다. 만료 전이면 그대로 쓸 수 있는 인증 수단이다.
- `alg`가 `none`이거나 기대와 다른 알고리즘인 토큰은 거부해야 한다(검증 라이브러리에서 허용 알고리즘을 고정).

## 참고
- RFC 7519 (JSON Web Token), RFC 7515 (JWS), RFC 7516 (JWE), RFC 7517 (JWK)
- Keycloak 문서: Server Administration Guide
