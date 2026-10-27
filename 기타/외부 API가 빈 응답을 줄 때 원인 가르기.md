# 외부 API가 빈 응답을 줄 때 원인 가르기

> 분류: 기타 · 최근 갱신: 2026-10-27

## 한 줄 요약
외부 서버(AI 등)가 200에 빈 본문을 주면 "실패"와 "해당 없음"이 구분되지 않는다. 원인을 가를 때는 테스트 요청이 실제 클라이언트 요청과 같은지부터 확인하고, 응답 시간과 입력 조합 비교로 좁힌다.

## 내용
- **"결과를 받지 못했습니다" 같은 공통 안내는 여러 원인의 합이다**: 연결 실패, 200이 아닌 응답, 200이지만 형식이 다른 응답(빈 본문 포함)이 같은 문구로 보인다. 서버 로그에 경우별로 다른 줄을 남겨 두면 로그 한 줄로 갈린다.
- **같은 서버·같은 사용자에서 다른 기능과 비교**: 같은 외부 서버의 다른 기능은 정상이고 한 기능만 빈 응답이면 연결·인증 문제가 아니라 그 기능의 처리 문제다.
- **테스트 스크립트가 실제 요청을 재현하는지 확인**: 화면을 거치지 않고 API에 본문만 직접 보내는 스크립트는, 화면이 덧붙이던 정보(예: 결재선·문맥 정보)를 빠뜨리기 쉽다. 요청 크기가 실제 화면 요청보다 눈에 띄게 작으면 의심한다. 빠진 입력으로 내린 결론("서버 처리 전체가 깨짐")은 틀릴 수 있고, 이미 공유했다면 정정해야 한다.
- **응답 시간이 단서다**: 수 ms 만에 빈 응답이면 서버가 입력 검사 단계에서 바로 끝낸 것, 1~2초 처리 뒤 빈 응답이면 처리를 하고도 결과를 비운 것이다. 정상 결과가 오는 경우의 시간과 나란히 놓고 본다.
- **입력 조합 표로 가른다**: "필수 정보 있음/없음" × "규칙에 해당하는 내용/무관한 내용"처럼 변수를 하나씩 바꿔 보낸다. 어느 칸에서만 빈 응답이 나오는지가 곧 원인이다.
- **진단 로그는 실제로 나간 요청을 찍는다**: 화면에서 만든 값이 서버를 거쳐 외부로 나갈 때 무엇이 붙었는지(헤더 이름, 본문 전문, 덧붙인 문자열)를 남겨야 "제대로 넘겼는가"에 답할 수 있다. 크기를 남긴다면 글자 수와 바이트 수를 따로 남긴다(한글은 다르다). API 키 같은 비밀값은 값 대신 유무·길이·해시 앞자리만 남긴다.
- **화면 상태와 서버 상태의 차이**: 화면에서 사용자가 고른 값이 저장 전까지 화면 변수에만 있고 서버는 DB 값만 읽는 구조라면, 저장 전에 하는 검사에는 사용자가 고른 값이 반영되지 않는다. 검사 요청에 화면의 현재 값을 함께 보내고, 없을 때만 저장된 값을 쓰게 한다.
- **구분 가능한 응답을 요청하거나 미리 막는다**: 외부 서버에 "필수 정보 없음", "해당 규칙 없음"을 빈 값 대신 구분된 응답으로 달라고 요청한다. 클라이언트에서는 필수 입력이 비었으면 호출 전에 막고 항목별로 안내한다. 빈 응답을 "해당 없음"으로 바꿔 보여 주는 방법은 서버 장애 때도 같은 문구가 나오므로 결정이 필요하다.

## 핵심 코드

### 1) 입력 조합 표 예
```
| 입력                       | 필수 정보 없음        | 필수 정보 있음                |
|----------------------------|-----------------------|-------------------------------|
| 규칙 A에 해당하는 본문     | 0.1초, 빈 응답        | 2.9초, 결과(FAIL)             |
| 규칙 B에 해당하는 본문     | 0.0초, 빈 응답        | 3.4초, 결과(FAIL)             |
| 어떤 규칙과도 무관한 본문  | 0.0초, 빈 응답        | 1.4초 처리 후 빈 응답         |
```

### 2) 경우별로 다른 로그 줄 남기기 (Java 8, HttpURLConnection)
```java
long t0 = System.nanoTime();
try {
    HttpURLConnection c = (HttpURLConnection) new URL(url).openConnection();
    c.setConnectTimeout(3000);
    c.setReadTimeout(30000);
    c.setRequestMethod("POST");
    c.setDoOutput(true);
    c.setRequestProperty("Content-Type", "application/json; charset=UTF-8");
    byte[] req = json.getBytes(StandardCharsets.UTF_8);
    log.debug("EXT_REQ chars={} bytes={}", json.length(), req.length);   // 글자 수와 바이트 수를 따로
    try (OutputStream o = c.getOutputStream()) { o.write(req); }

    int code = c.getResponseCode();
    InputStream in = code >= 400 ? c.getErrorStream() : c.getInputStream();
    String body = (in == null) ? "" : readAll(in);
    long ms = (System.nanoTime() - t0) / 1_000_000;
    if (code != 200)                log.warn("EXT_STATUS code={} ms={} head={}", code, ms, head(body));
    else if (body.trim().isEmpty()) log.warn("EXT_EMPTY ms={}", ms);              // 200 + 빈 본문
    else if (!looksValid(body))     log.warn("EXT_FORMAT ms={} head={}", ms, head(body));
    else                            log.info("EXT_OK ms={} len={}", ms, body.length());
} catch (IOException e) {
    log.warn("EXT_CONNECT ms={} err={}", (System.nanoTime() - t0) / 1_000_000, e.toString());
}
```
```java
static String readAll(InputStream in) throws IOException {
    ByteArrayOutputStream b = new ByteArrayOutputStream();
    byte[] buf = new byte[4096];
    int n;
    while ((n = in.read(buf)) != -1) b.write(buf, 0, n);
    in.close();
    return new String(b.toByteArray(), StandardCharsets.UTF_8);
}
static String head(String s) { return s.length() <= 200 ? s : s.substring(0, 200) + "..."; }
```

### 3) 비밀값은 유무·길이·해시 앞자리만 로그에
```java
static String maskSecret(String key) throws NoSuchAlgorithmException {
    if (key == null || key.isEmpty()) return "none";
    byte[] h = MessageDigest.getInstance("SHA-256").digest(key.getBytes(StandardCharsets.UTF_8));
    return "len=" + key.length() + " sha256=" + String.format("%02x%02x%02x", h[0], h[1], h[2]);
}
```

### 4) 입력 조합을 한 번에 돌리는 시험 스크립트 (bash + curl)
```bash
# 조합마다 요청 본문 파일을 준비해 둔다. 출력 시각으로 서버 로그와 맞춘다
for f in A_noinfo.json A_info.json B_noinfo.json B_info.json none_info.json; do
  printf '%s %-16s ' "$(date '+%H:%M:%S')" "$f"
  curl -s -o "out_$f" -w 'status=%{http_code} time=%{time_total}s size=%{size_download}\n' \
       -H 'Content-Type: application/json' --data-binary "@$f" "https://<서버>/api/check"
done
```

### 5) 화면: 현재 값을 함께 보내고, 필수 입력이 비면 호출 전에 막기
```javascript
function runCheck() {
  var payload = { body: getBody(), reviewers: currentReviewers() };   // 저장 전 화면 값도 보낸다
  if (!payload.reviewers || payload.reviewers.length === 0) {
    alert("검토자를 먼저 지정하세요.");                                // 빈 응답 대신 미리 안내
    return;
  }
  fetch("/api/check", { method: "POST", headers: { "Content-Type": "application/json" },
                        body: JSON.stringify(payload) })
    .then(function (r) { if (!r.ok) throw new Error("status " + r.status); return r.text(); })
    .then(function (t) { if (!t.trim()) showInfo("판정 결과가 없습니다."); else render(JSON.parse(t)); })
    .catch(function (e) { console.error(e); alert("검사 서버에 연결하지 못했습니다."); });  // 실패와 "결과 없음"을 나눈다
}
```

## 주의할 점
- 저장분이 몇 분치뿐인 서버 로그는 재현 직후 바로 떠 둔다. 테스트 스크립트는 요청 시각을 출력해 로그와 맞춘다.
- 본문 전문이나 사용자 정보를 남기는 진단 로그는 테스트 서버에서만 쓰고, 확인 후 소스를 되돌리고 다시 배포한다.
- 내용이 의미 없는 입력에도 엉뚱한 규칙으로 판정하는 결과가 오면, 전달은 정상이고 외부 서버의 판정 문제일 수 있다. 같은 입력으로 두 번 돌려 재현성을 확인한다.
