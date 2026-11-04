# 화면 제한과 서버 측 접근 확인(IDOR)

> 분류: 인증·보안 · 최근 갱신: 2026-11-03

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

### 권한은 ID로 판정하고 파일은 경로로 고르는 다운로드
- 첨부 다운로드 주소가 문서 ID와 파일 경로(`filePath`)를 함께 받는 구조가 흔하다. 권한은 문서 ID로 보는데 실제로 내려줄 파일은 경로로 고르면 구멍이 생긴다. 볼 수 있는 문서의 ID에 다른 문서의 경로를 붙여도 받아진다.
- 문서 ID 같은 권한 판정용 파라미터를 빼면 판정 분기를 아예 건너뛰는 구현도 있다. "값이 있으면 검사"는 "값이 없으면 통과"와 같다.
- 판정 결과를 보지 않는 다른 분기(미리보기·뷰어 변환 등)가 있는지도 확인한다.
- 고치는 방향: 클라이언트에서 받는 것은 `문서 ID + 첨부 순번`뿐이다. 서버가 그 둘로 첨부 행을 찾아 경로와 파일 이름을 DB에서 읽는다. 권한 판정은 같은 ID로 한다. 첨부를 찾는 데 필요 없는 상위 ID(게시판 ID 등)가 권한 판정에 필요하면, 클라이언트에서 받지 말고 서버가 조회한다. 받더라도 조회한 값과 같은지만 확인한다.

### 권한 판정이 흩어진 시스템에서 생기는 구멍
오래된 시스템의 권한 체크를 전수조사하면 자주 나오는 모양이다.
- **판정은 1단계, 데이터는 2·3단계**: 팝업을 여는 요청에서만 판정하고 결과(`pass`)를 화면에 넘기면, 막는 것은 화면 JS다. 실제로 파일 목록을 주거나 zip을 만들어 내려주는 다음 API를 직접 부르면 통과한다. 판정은 데이터를 돌려주는 API 안에서 다시 한다. 판정 전에 파일 복사 같은 작업을 먼저 하는 것도 피한다.
- **통과 플래그와 클라이언트 값**: `fromLink=Y`, `skipCheck=Y` 같은 요청 파라미터로 판정을 건너뛰는 분기, 사용자 ID·회사 ID를 세션이 아니라 요청에서 받는 API, "값이 비면 검사 생략" 분기는 모두 사용자가 직접 켤 수 있는 스위치다.
- **대상을 못 찾으면 통과**: 판정 함수가 문서를 조회하지 못했을 때 허용으로 끝나면, 상태(진행·완료)에 맞지 않는 판정 함수를 부르는 경로에서 검사가 사실상 꺼진다.
- **경로마다 다른 판정 조합**: 같은 문서를 여는 경로(본문 화면 종류별, 부속 탭, 첨부, 외부 API, 모바일)가 각각 다른 함수·스위치·관리자 우회 조건을 쓰면 같은 사람이 경로에 따라 다른 결과를 받는다. 화면을 하나로 합치는 리팩터링에서 각자의 검사를 분기로 그대로 옮기면 이 차이가 굳어진다.
- **목록과 열기 기준 불일치**: 목록 쿼리 조건과 열기 판정이 다른 스위치를 보면 "목록에는 없는데 열림", "목록에는 있는데 거부"가 생긴다.
- **관리자 API**: 관리자 화면 진입 때만 역할을 확인하면, 로그인한 일반 사용자가 관리자 AJAX를 직접 불러도 통과한다. IP 대역만 보는 인터셉터는 역할 확인이 아니다.
- 고치는 방향: 판정 메서드 하나(`canAccess(docId, user, action)`)를 두고 모든 경로가 그것만 쓰게 한다. 사용자 정보는 세션·토큰에서만 얻고, 통과 플래그는 없앤다.
- 전수조사 요령: 인터셉터(로그인·역할) → 컨트롤러(판정 함수 직접 호출) → 서비스 → 목록 쿼리 조건 순으로 계층을 나눠, URL마다 "어느 계층에서 무엇을 보는지" 표를 만든다. 변경성 API(삭제·저장) 중 검사가 없는 것은 따로 목록으로 뺀다. 코드만 읽은 결과는 실제 재현 전까지 "추정"으로 적는다.

## 핵심 코드
### 0) 판정을 한 곳으로: 데이터를 주는 API마다 같은 메서드로 재판정
```java
public enum Action { LIST, READ, EXPORT, APPROVE, MANAGE }

@Service
public class DocAccessService {
    public boolean canAccess(String docId, LoginUser user, Action action) {
        Doc doc = docRepository.findById(docId);
        if (doc == null) return false;                       // 못 찾으면 거부 (통과 아님)
        if (user.hasRole("ADMIN") && action != Action.APPROVE) return true;  // 관리자 우회 조건은 여기 한 곳
        boolean involved = lineRepository.isInvolved(docId, user.getId());
        boolean open = doc.isPublicFor(user) && user.getSecurityLevel() <= doc.getSecurityLevel();
        return doc.isInProgress() ? involved || open : involved || open || readerRepository.isReader(docId, user.getId());
    }
}

// 팝업용 확인 API뿐 아니라 실제 zip을 내려주는 API에서도 다시 판정
@PostMapping("/docs/{docId}/export")
public void export(@PathVariable String docId, HttpSession session, HttpServletResponse res) throws IOException {
    LoginUser user = (LoginUser) session.getAttribute("loginUser");   // 요청 파라미터의 userId는 쓰지 않음
    if (user == null || !docAccessService.canAccess(docId, user, Action.EXPORT)) { res.sendError(404); return; }
    exportService.writeZip(docId, res.getOutputStream());              // 파일 경로는 서버가 docId로 찾음
}
```

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

### 4) 첨부 다운로드: ID와 순번만 받고 경로는 서버가 찾기
```java
@GetMapping("/items/attach/download")
public void download(@RequestParam String itemId, @RequestParam int seq,
                     HttpSession session, HttpServletResponse res) throws IOException {
    String userId = (String) session.getAttribute("loginUserId");
    if (userId == null) { res.sendError(401); return; }
    Item item = itemService.findById(itemId);                 // 상위 ID(boardId)는 서버가 조회
    if (item == null || !accessService.canRead(userId, item.getBoardId(), itemId)) {
        res.sendError(404); return;                           // 없음·권한 없음을 같은 응답으로
    }
    Attachment a = attachService.find(itemId, seq);           // 경로·이름은 DB에서
    if (a == null) { res.sendError(404); return; }
    res.setHeader("X-Content-Type-Options", "nosniff");
    fileSender.send(res, a.getStoredPath(), a.getFileName());  // 클라이언트가 보낸 경로는 쓰지 않음
}
```

### 5) 다른 사용자로 직접 호출해 확인
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
