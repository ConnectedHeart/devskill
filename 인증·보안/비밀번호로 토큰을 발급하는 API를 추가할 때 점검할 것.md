# 비밀번호로 토큰을 발급하는 API를 추가할 때 점검할 것

> 분류: 인증·보안 · 최근 갱신: 2026-10-27

## 한 줄 요약
화면 로그인과 별도로 아이디·비밀번호를 받아 토큰을 주는 API를 만들면, 화면 로그인 안에 묶여 있던 보안 검사가 빠지기 쉽다. 어떤 검사를 옮기고 뺄지, 로그와 전송 방식을 함께 정해야 한다.

## 내용
- **세션 기반 토큰의 수명**: 토큰이 "서버 세션 ID를 암호화한 값"이면 토큰 자체에는 만료 정보가 없고 세션 저장소의 만료를 따른다. API 호출이 세션의 마지막 접근 시각을 갱신하지 않게 해 두었다면 발급 후 고정 수명이 되고, 리프레시가 없으니 만료되면 다시 발급받아야 한다. 발급할 때마다 세션 행이 쌓이므로 정리 주기를 확인한다.
- **화면 로그인 검사 중 옮길지 정할 것**

| 검사 | 옮기지 않으면 |
|---|---|
| 실패 횟수·계정 잠금 | 인증 없는 엔드포인트라 무차별 대입 가능 |
| 2단계 인증(OTP·FIDO) | 비밀번호만으로 우회됨 (서버 간 호출용이면 일부러 뺄 수 있음 — 결정으로 남긴다) |
| 외부 디렉터리(AD) 연동 | 해당 사용자는 로그인 실패 |
| 정지·특수 계정 차단 | 화면에서 막힌 계정이 API로 통과 |
| IP 접근 제한 | 호출자가 서버면 의미가 달라짐 |
| 비밀번호 만료·첫 로그인 변경 강제 | API로는 그대로 발급 |

  실패 횟수와 잠금은 화면 로그인과 같은 저장값을 공유하는 것이 자연스럽다. 대신 API로 틀린 횟수만큼 화면 로그인도 잠긴다.
- **관리자 계정**: 인증 없는 경로에서 관리자 아이디로 틀린 비밀번호를 반복하면 관리자 계정을 잠글 수 있는 통로가 하나 늘어난다. 관리자 토큰 발급을 허용할지 정한다.
- **로그**: 화면 접속 이력 테이블에 API 발급을 섞으면 통계가 흐려지고 로그아웃 시각이 채워지지 않는다. 실패만 서버 로그에 아이디·IP로 남기는 방식이 깔끔하다. 존재하지 않는 사용자일 때는 입력한 아이디를 남기지 않는다(아이디 칸에 비밀번호를 잘못 넣는 경우가 있다). 앞단 서버가 대신 호출하면 그 서버 IP가 찍힌다.
- **전송**: JSON 본문 평문은 HTTPS에만 의존한다. 서버 간 호출이고 HTTPS가 보장되면 흔한 방식이다. 화면처럼 RSA로 암호화해 받으려면 호출자가 공개키를 받아 암호화해야 하므로 규격을 맞춰야 한다.
- **쿼리스트링 금지**: 접근 로그가 경로와 쿼리스트링을 기록하는 경우가 많아 `?password=...`로 보내면 비밀번호가 로그에 평문으로 남는다. 본문으로만 받게 하고, 알 수 없는 쿼리 파라미터를 400으로 거르는 필터가 있으면 이를 막아 준다.

## 핵심 코드
### 1) 컨트롤러: 본문으로만 받고 쿼리스트링은 거부
```java
@PostMapping(value = "/api/token", consumes = MediaType.APPLICATION_JSON_VALUE)
public ResponseEntity<?> token(@RequestBody TokenRequest body, HttpServletRequest req) {
    if (req.getQueryString() != null) {
        return ResponseEntity.badRequest().build();        // ?password=... 가 로그에 남는 경로 차단
    }
    TokenResponse t = authService.issueToken(body.getUsername(), body.getPassword(), req.getRemoteAddr());
    if (t == null) return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
    return ResponseEntity.ok()
        .header("Cache-Control", "no-store")               // 토큰 응답은 캐시 금지
        .header("Pragma", "no-cache")
        .body(t);
}
```

### 2) 서비스: 화면 로그인의 검사를 같은 저장값으로 옮긴다
```java
public TokenResponse issueToken(String id, String pw, String ip) {
    if (isBlank(id) || isBlank(pw)) return null;
    Account a = accountRepo.find(id);
    if (a == null) {
        log.info("token issue failed: unknown user ip={}", ip);    // 입력 아이디는 남기지 않는다
        return null;
    }
    if (a.isLocked() || a.isSuspended() || a.isPasswordExpired()) {
        log.info("token issue failed: blocked account id={} ip={}", id, ip);
        return null;
    }
    if (!passwordChecker.matches(a, pw)) {                  // AD 사용자면 LDAP bind
        accountRepo.increaseFailCount(id);                  // 화면 로그인과 같은 카운터
        log.info("token issue failed: bad password id={} ip={}", id, ip);
        return null;
    }
    accountRepo.resetFailCount(id);
    // 2단계 인증 생략: 서버 간 호출 전용이라 일부러 뺌 (결정 근거를 여기에 남긴다)
    return tokenService.create(a);
}
```

### 3) 접근 로그에서 쿼리스트링 빼기 (Tomcat server.xml)
```xml
<!-- %r(요청 줄 전체) 대신 %m %U %H: 쿼리스트링을 기록하지 않는다 -->
<Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
       prefix="access_log" suffix=".txt"
       pattern="%h %t &quot;%m %U %H&quot; %s %b %D" />
```

### 4) 호출과 잠금 동작 확인
```bash
curl -X POST "https://api.example.com/api/token" \
  -H "Content-Type: application/json" \
  -d '{"username":"user1","password":"<비밀번호>"}'
# 응답 예: { "accessToken": "...", "tokenType": "Bearer", "expiresInSeconds": 3600 }

# 테스트 계정으로 틀린 비밀번호 반복 → 잠금 후 화면 로그인도 잠기는지 확인
for i in 1 2 3 4 5 6; do
  curl -s -o /dev/null -w "%{http_code}\n" -X POST "https://api.example.com/api/token" \
    -H "Content-Type: application/json" -d '{"username":"testuser","password":"wrong"}'
done
```

## 주의할 점
- 보안 검사를 빼는 결정(2단계 인증 생략 등)은 코드 주석과 문서에 이유와 함께 남긴다.
- 실제 호출로 잠금·로그 동작을 확인하기 전에는 배포하지 않는다.

## 참고
- OWASP Authentication Cheat Sheet
- OAuth 2.0 Resource Owner Password Credentials (RFC 6749 4.3) — 권장되지 않는 이유 포함
