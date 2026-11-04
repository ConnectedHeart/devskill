# URL의 ID 파라미터 숨기기와 서명 링크 토큰

> 분류: 인증·보안 · 최근 갱신: 2026-11-03

## 한 줄 요약
브라우저가 서버에 보내는 값은 사용자 본인에게서 숨길 수 없다. "ID가 보인다"는 지적에는 숨기기가 아니라 서버 권한 판정으로 대응하고, 링크 공유·번호 바꿔치기를 막고 싶을 때 서명(또는 DB 저장) 링크 토큰을 쓴다.

## 내용
### 완전히 숨길 수 없는 이유
- URL, POST 본문, 헤더, 쿠키 어디에 넣어도 개발자 도구 Network 탭이나 로컬 프록시로 사용자 본인은 다 볼 수 있다. HTTPS는 중간에서 엿보는 제3자만 막는다.
- 서버 세션에만 "지금 보는 문서"를 두는 방식은, 같은 세션을 쓰는 여러 창이 서로 섞이고 화면 안의 후속 요청(댓글·첨부 등)을 모두 바꿔야 해서 현실적이지 않다.

### 목표별로 할 수 있는 것
| 목표 | 방법 | 한계 |
|---|---|---|
| 주소창에 안 보이게 | POST로 열기, 열린 뒤 `history.replaceState`로 쿼리 지우기 | Network 탭에는 보임. 처음 GET은 access log에 남음 |
| 봐도 다른 번호로 바꿔 넣지 못하게 | ID 대신 서명·암호화 값이나 랜덤 토큰 | 값 자체는 보임 |
| 남이 받은 링크로 못 열게 | 사용자에 묶고 만료 시간을 둔 토큰 | 미리 보내는 메일 링크는 별도 처리 필요 |

- 보안 점검 대응의 본질은 "ID를 바꿔 넣어도 권한 없는 대상은 열리지 않는다"를 서버 판정으로 보이는 것이다([화면 제한과 서버 측 접근 확인(IDOR)](화면 제한과 서버 측 접근 확인(IDOR).md)).

### 주소창 숨김 + 새로고침 처리
- 열린 화면이 `replaceState`로 쿼리를 지우면 새로고침 때 ID 없이 요청이 간다. 파라미터를 `sessionStorage`(탭별)에 넣어 두고, ID 없는 요청에는 작은 중계 페이지가 그 값으로 POST를 다시 보내게 한다.
- 오류·권한 없음 화면처럼 그 스크립트가 없는 화면에서는 주소가 그대로 남는다.

### 링크 토큰 두 방식
- **DB 저장 랜덤 토큰**: 열 때마다 서버가 권한을 확인한 뒤 랜덤 값을 만들고 "토큰·대상 ID·사용자·만료"를 저장한다. 일회용도 가능하다. 서버가 여러 대면 메모리가 아니라 DB(공용 저장소)에 둬야 하므로 테이블이 필요하다.
- **서명 토큰(무상태)**: "대상 ID·사용자 ID·발급 시각"을 서버 비밀 키로 HMAC 서명(ID까지 숨기려면 암호화)해 링크에 넣는다. 서버끼리 같은 키만 공유하면 되고 테이블이 필요 없다. 발급 시각이 들어가 링크가 매번 바뀐다. 대신 일회용으로는 못 만들고 만료 전에는 재사용된다.

### 어느 방식이든 남는 문제
- 일회용으로 만들면 새로고침이 실패한다. 짧은 만료가 현실적이다.
- 메일·알림처럼 미리 만들어 보내는 링크는 누를 때마다 새로 만들 수 없다. ID 링크로 두거나, 그 링크가 권한 확인 후 토큰을 발급해 다시 넘기는 단계를 거친다.
- 화면이 열린 뒤의 후속 요청은 여전히 ID를 보낸다. 토큰으로 바뀌는 것은 처음 여는 주소뿐이다.

## 핵심 코드
### 1) 서명 링크 만들기·검증 (Java 8, HMAC-SHA256)
```java
// 키는 코드·저장소에 두지 말고 환경 변수나 비밀 저장소에서 읽는다
private final byte[] key = System.getenv("LINK_SIGN_KEY").getBytes(StandardCharsets.UTF_8);

public String issue(String docId, String userId) {
    String payload = docId + "|" + userId + "|" + System.currentTimeMillis();
    return b64(payload.getBytes(StandardCharsets.UTF_8)) + "." + b64(hmac(payload));
}

public String verify(String token, String userId, long ttlMillis) {   // 실패하면 null
    String[] p = token.split("\\.");
    if (p.length != 2) return null;
    String payload = new String(Base64.getUrlDecoder().decode(p[0]), StandardCharsets.UTF_8);
    if (!MessageDigest.isEqual(hmac(payload), Base64.getUrlDecoder().decode(p[1]))) return null; // 상수 시간 비교
    String[] f = payload.split("\\|");
    if (!f[1].equals(userId)) return null;                                  // 다른 사용자 차단
    if (System.currentTimeMillis() - Long.parseLong(f[2]) > ttlMillis) return null; // 만료
    return f[0];                                                            // docId → 이후 권한 판정은 그대로
}

private byte[] hmac(String s) {
    try {
        Mac mac = Mac.getInstance("HmacSHA256");
        mac.init(new SecretKeySpec(key, "HmacSHA256"));
        return mac.doFinal(s.getBytes(StandardCharsets.UTF_8));
    } catch (GeneralSecurityException e) { throw new IllegalStateException(e); }
}
private static String b64(byte[] b) { return Base64.getUrlEncoder().withoutPadding().encodeToString(b); }
```

### 2) DB 저장 토큰 테이블
```sql
CREATE TABLE view_link_token (
    token       VARCHAR(64) PRIMARY KEY,   -- SecureRandom 32바이트 hex
    doc_id      VARCHAR(40) NOT NULL,
    user_id     VARCHAR(40) NOT NULL,
    expires_at  TIMESTAMP   NOT NULL
);
-- 검증: 토큰·사용자·만료가 모두 맞을 때만
SELECT doc_id FROM view_link_token
 WHERE token = ? AND user_id = ? AND expires_at > CURRENT_TIMESTAMP;
```

### 3) 주소창 숨김과 새로고침 재전송 (브라우저)
```javascript
// 열린 화면: 쿼리를 탭 저장소로 옮기고 주소창에서 지움
if (location.search) {
  sessionStorage.setItem('viewParams', location.search.substring(1));
  history.replaceState(null, '', location.pathname);
}
// ID 없이 들어온 요청을 받는 중계 페이지: 저장한 값으로 POST 재전송
const form = document.createElement('form');
form.method = 'post'; form.action = '/docs/view';
new URLSearchParams(sessionStorage.getItem('viewParams') || '').forEach((v, k) => {
  const i = document.createElement('input'); i.type = 'hidden'; i.name = k; i.value = v; form.appendChild(i);
});
document.body.appendChild(form); form.submit();
```

## 주의할 점
- 토큰을 써도 서버의 열람 권한 판정은 그대로 해야 한다. 토큰은 판정을 대신하지 않는다.
- 서명 키를 저장소나 설정 파일에 평문으로 넣지 않는다. 키를 바꾸면 이미 발급한 링크는 모두 무효가 된다.
- 서명 비교는 `equals`가 아니라 상수 시간 비교(`MessageDigest.isEqual`)로 한다.

## 참고
- OWASP Cheat Sheet: Insecure Direct Object Reference Prevention
- RFC 2104 (HMAC)
