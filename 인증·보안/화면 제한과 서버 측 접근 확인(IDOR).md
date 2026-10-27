# 화면 제한과 서버 측 접근 확인(IDOR)

> 분류: 인증·보안 · 최근 갱신: 2026-10-27

## 한 줄 요약
"이 버튼은 작성 화면에서만 보인다"는 보안이 아니다. 클라이언트가 보낸 ID로 서버가 파일·데이터를 꺼내 주는 API는 서버에서 "요청자가 이 대상을 볼 수 있는가"를 확인해야 한다.

## 내용
### IDOR(Insecure Direct Object Reference)
- 요청에 들어 있는 식별자(문서 번호, 파일 ID 등)만 믿고 서버가 그 대상을 돌려주는 결함이다.
- 로그인한 사용자라면 누구나 개발자 도구나 스크립트로 주소와 파라미터를 바꿔 직접 호출할 수 있다. 화면에서 버튼을 숨기거나 특정 화면에서만 호출하도록 만든 것은 막는 장치가 아니다.
- 식별자가 순번(…0800, …0801)이면 추측이 쉬워 위험이 커진다.

### 어떤 API에 확인이 필요한가
- **서버가 ID로 데이터를 꺼내는 API**에만 필요하다. 예: 문서 번호를 받아 서버가 첨부 파일을 찾아 AI 분석 결과를 돌려주는 기능. 다운로드 권한이 없는 사람도 "요약·분석 결과"라는 형태로 내용을 얻을 수 있다.
- 클라이언트가 내용 자체를 보내고 서버는 그걸 처리만 하는 API(맞춤법 검사처럼 본문을 직접 보내는 경우)는 꺼낼 대상이 없으므로 이 확인이 필요 없다.

### 확인 기준 고르기
- 가장 좁은 기준: "요청자가 그 문서의 작성자인가". 작성 화면 전용 기능이면 충분하다.
- 더 맞는 기준: "요청자가 그 대상을 열람할 권한이 있는가". 결재자·수신자도 쓰는 기능이 되면 기존 열람 권한 판정을 재사용하는 편이 맞다.
- **아직 저장되지 않은 대상**은 작성자 정보가 DB에 없어 소유 판정이 실패하기 쉽다. "정보가 없으면 통과"로 바꾸면 남의 작성 중 데이터와 존재하지 않는 ID까지 통과하므로 위험하다. 대신 비교할 수 있는 다른 값(예: 첨부를 올린 사용자)을 쓰거나, 임시 ID를 발급할 때 소유자를 기록한다.
- 비교에 쓰는 값이 **클라이언트가 보낸 값을 그대로 저장한 것**이면 조작될 수 있다. 서버가 로그인 정보로 채우는 값인지 확인한다.

## 핵심 코드
### 1) 컨트롤러: ID로 꺼내기 전에 열람 권한 확인
```java
@GetMapping("/docs/{docId}/ai-summary")
public ResponseEntity<?> summary(@PathVariable String docId, HttpSession session) {
    String userId = (String) session.getAttribute("loginUserId");  // 요청 파라미터가 아니라 로그인 정보
    if (userId == null) return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
    if (!docAccessService.canRead(userId, docId)) {                // 기존 열람 권한 판정을 재사용
        log.warn("access denied reason=no_read_permission user={} doc={}", userId, docId);
        return ResponseEntity.status(HttpStatus.FORBIDDEN).build(); // 존재 여부를 숨기려면 404
    }
    return ResponseEntity.ok(summaryService.summarize(docId));
}
```

### 2) 열람 권한 판정 쿼리 (작성자 또는 열람자, 같은 테넌트)
```sql
SELECT COUNT(*)
  FROM document d
 WHERE d.doc_id = ?
   AND d.tenant_id = ?                       -- 최소 범위 제한
   AND (d.writer_id = ?
        OR EXISTS (SELECT 1 FROM document_reader r
                    WHERE r.doc_id = d.doc_id AND r.user_id = ?))
```

### 3) 저장 전 대상: 임시 ID를 발급할 때 소유자를 기록
```java
String tempId = UUID.randomUUID().toString();          // 순번 대신 추측 불가한 값
jdbc.update("INSERT INTO draft_owner (temp_id, owner_id, created_at) VALUES (?, ?, ?)",
    tempId, userId, new Timestamp(System.currentTimeMillis()));

// 확인: "정보가 없으면 통과"가 아니라 "행이 있고 소유자가 같을 때만 통과"
Integer n = jdbc.queryForObject(
    "SELECT COUNT(*) FROM draft_owner WHERE temp_id = ? AND owner_id = ?",
    Integer.class, tempId, userId);
boolean allowed = n != null && n > 0;
```

### 4) 다른 사용자로 직접 호출해 확인
```bash
# 권한 없는 사용자의 세션 쿠키로 남의 문서 ID를 넣어 호출 → 403(또는 404)이어야 정상
curl -i -b "<세션 쿠키 이름>=<권한 없는 사용자 세션>" "https://example.com/docs/<남의 문서 ID>/ai-summary"
```

## 주의할 점
- 확인을 없애기로 결정했다면 최소한 범위(같은 회사·테넌트 등)는 서버에서 제한하고, 남는 위험을 코드 주석과 문서에 적어 둔다.
- 거절할 때는 사유와 대상 ID를 서버 로그에 남겨야 나중에 "권한 판정 때문에 막혔는지"를 로그로 확정할 수 있다.

## 참고
- OWASP Top 10: Broken Access Control
- OWASP Cheat Sheet: Insecure Direct Object Reference Prevention
