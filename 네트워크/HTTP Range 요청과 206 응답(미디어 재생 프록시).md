# HTTP Range 요청과 206 응답(미디어 재생 프록시)

> 분류: 네트워크 · 최근 갱신: 2026-10-27

## 한 줄 요약
브라우저 `<audio>`/`<video>`는 파일을 통째로 받지 않고 `Range` 헤더로 구간만 요청하며, 서버가 `206 Partial Content`와 `Content-Range`로 답해야 탐색(seek)이 된다. 중간에 프록시 서버가 있으면 이 헤더와 상태코드를 그대로 전달해야 한다.

## 내용
### 요청과 응답 형태
```
요청:  Range: bytes=1048576-2097151        ← 1MB 지점부터 1MB
응답:  206 Partial Content
       Content-Range: bytes 1048576-2097151/52428800
       Accept-Ranges: bytes
       Content-Length: 1048576
```
- 재생 위치 이동, 중간 구간부터 재생, "이 문장부터 듣기" 같은 기능이 전부 Range 요청이다.
- 범위가 파일 크기를 벗어나면 `416 Range Not Satisfiable`로 답한다.
- iOS Safari는 서버가 206을 주지 않으면 아예 재생하지 않거나 탐색이 안 된다. 데스크톱 브라우저는 200 전체 응답으로도 재생은 되지만 탐색이 느리거나 안 된다.

### 2단 구조(화면 서버 → 파일 서버)에서 생기는 문제
- 모바일 화면 서버가 DB·파일에 직접 붙지 못하고 내부 API 서버를 거치는 구조라면, 화면 서버가 음성 파일을 중계해야 한다.
- 흔한 기존 다운로드 프록시는 ① 내부 API가 파일을 base64 JSON으로 통째로 주고 화면 서버가 디코딩해 보내거나, ② 스트리밍은 하지만 `Content-Disposition`·`Content-Type`만 넘기고 상태코드·`Content-Range`는 버린다. 둘 다 Range가 안 된다. 수백 MB 파일을 base64로 받으면 메모리도 문제다.

### 만들어야 하는 것
1. 브라우저의 `Range` 헤더를 내부 API 호출에 그대로 전달한다.
2. 내부 API(파일 서버)는 Range를 처리해 206/200/416과 `Content-Range`, `Content-Length`, `Accept-Ranges`, `Content-Type`을 돌려준다. 이미 PC용으로 Range를 처리하는 엔드포인트가 있다면 권한 검사만 붙여 재사용한다.
3. 프록시는 상태코드와 위 헤더를 그대로 되돌려 주고, 본문은 스트림으로 복사한다(메모리에 올리지 않는다).

## 핵심 코드
### 1) 파일 서버: 단일 구간 Range 처리 (206 / 200 / 416, Java 8)
```java
long size = file.length();
long start = 0, end = size - 1;
boolean partial = false;
String range = req.getHeader("Range");
resp.setHeader("Accept-Ranges", "bytes");
resp.setContentType(contentType);                       // 예: audio/mpeg
if (range != null && range.startsWith("bytes=") && !range.contains(",")) {   // 여러 구간은 200 전체로
    String[] se = range.substring(6).trim().split("-", 2);
    try {
        if (se[0].isEmpty()) {                          // bytes=-500 : 마지막 500바이트
            start = Math.max(0, size - Long.parseLong(se[1]));
        } else {
            start = Long.parseLong(se[0]);
            if (se.length > 1 && !se[1].isEmpty()) end = Math.min(Long.parseLong(se[1]), size - 1);
        }
        partial = true;
    } catch (NumberFormatException e) { start = 0; end = size - 1; }   // 형식 오류는 무시하고 전체
    if (partial && (start >= size || start > end)) {
        resp.setStatus(416);
        resp.setHeader("Content-Range", "bytes */" + size);
        return;
    }
}
if (partial) {
    resp.setStatus(206);
    resp.setHeader("Content-Range", "bytes " + start + "-" + end + "/" + size);
}
long len = end - start + 1;
resp.setHeader("Content-Length", String.valueOf(len));
try (RandomAccessFile raf = new RandomAccessFile(file, "r")) {
    raf.seek(start);
    OutputStream out = resp.getOutputStream();
    byte[] buf = new byte[8192];
    long remaining = len;
    int n;
    while (remaining > 0 && (n = raf.read(buf, 0, (int) Math.min(buf.length, remaining))) != -1) {
        out.write(buf, 0, n);
        remaining -= n;
    }
}
```

### 2) 중계(프록시): Range 전달, 상태코드·헤더 복사, 스트림 복사 (Java 8, HttpURLConnection)
```java
HttpURLConnection up = (HttpURLConnection) new URL(upstreamUrl).openConnection();
up.setConnectTimeout(5000);
up.setReadTimeout(30000);
String range = req.getHeader("Range");
if (range != null) up.setRequestProperty("Range", range);      // 브라우저 Range 그대로
try {
    int status = up.getResponseCode();                          // 206 / 200 / 416
    resp.setStatus(status);
    for (String name : new String[] {"Content-Range", "Content-Length", "Accept-Ranges", "Content-Type"}) {
        String v = up.getHeaderField(name);
        if (v != null) resp.setHeader(name, v);
    }
    InputStream in = status >= 400 ? up.getErrorStream() : up.getInputStream();
    if (in == null) return;
    OutputStream out = resp.getOutputStream();
    byte[] buf = new byte[8192];
    int n;
    while ((n = in.read(buf)) != -1) out.write(buf, 0, n);     // 브라우저가 끊으면 IOException
} finally {
    up.disconnect();                                            // 위쪽 연결도 닫아 남은 파일을 받지 않는다
}
```

### 3) 브라우저: 탭 이벤트 안에서 바로 재생 (iOS Safari)
```javascript
var audio = document.getElementById('player');
audio.src = mediaUrl;                 // 주소는 미리 정해 둔다
playBtn.addEventListener('click', function () {
  audio.currentTime = startSec;       // "이 문장부터 듣기"
  audio.play();                       // 비동기 처리 뒤에 부르면 iOS가 막는다
});
```

### 4) curl로 206·416 확인
```bash
curl -s -D - -o /dev/null -H "Range: bytes=0-1023" https://example.com/media/sample.mp3   # 206, Content-Range
curl -s -D - -o /dev/null -H "Range: bytes=999999999-" https://example.com/media/sample.mp3 # 416
curl -s -D - -o /dev/null https://example.com/media/sample.mp3                            # 200, Accept-Ranges
```

### 5) 참고: RestTemplate 개념 코드 (Java 9+ 메소드 포함)
```java
// 프록시 쪽 개념 코드
HttpHeaders h = new HttpHeaders();
String range = request.getHeader("Range");
if (range != null) h.set("Range", range);

ResponseEntity<Resource> up = restTemplate.exchange(url, GET, new HttpEntity<>(h), Resource.class);

response.setStatus(up.getStatusCodeValue());            // 206 / 200 / 416
copyHeader(up, response, "Content-Range");
copyHeader(up, response, "Content-Length");
copyHeader(up, response, "Accept-Ranges");
copyHeader(up, response, "Content-Type");
try (InputStream in = up.getBody().getInputStream()) {
    in.transferTo(response.getOutputStream());
}
```

## 주의할 점
- 압축 필터(gzip)가 끼면 `Content-Length`와 범위가 어긋난다. 미디어 경로는 압축에서 제외한다.
- 위 예시처럼 RestTemplate으로 받으면, 브라우저가 구간을 옮기며 앞 요청을 끊어도 프록시가 위쪽 응답을 끝까지 읽느라 남은 파일 전체를 내려받는 일이 생길 수 있다. 이럴 때는 HttpURLConnection(또는 `execute` + 직접 스트림 처리)으로 위쪽 스트림을 바로 복사하고, 클라이언트가 끊으면 위쪽 연결도 닫는다.
- `InputStream.transferTo`는 Java 9부터 있다. Java 8에서는 버퍼 루프로 복사한다.
- PC와 중계 서버가 같은 Range 처리 코드를 쓰도록 공용 클래스로 빼 두면 206·416 처리가 어긋나지 않는다.
- iOS Safari는 사용자 탭 이벤트 안에서 바로 `play()`를 부르지 않으면 재생을 막는다. 비동기 처리(주소 조회 등)를 끝낸 뒤 재생하면 막히므로, 주소를 미리 정해 두고 탭 핸들러에서 바로 재생한다.
- 인증을 쿠키·세션으로 하는 경우 `<audio src>` 요청에도 같은 세션이 붙는지 확인한다. 토큰을 URL에 넣는 방식은 피한다.

## 참고
- RFC 9110 (HTTP Semantics): Range Requests, 206 Partial Content, 416
- MDN: HTTP range requests
