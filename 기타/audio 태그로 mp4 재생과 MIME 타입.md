# audio 태그로 mp4 재생과 MIME 타입

> 분류: 기타 · 최근 갱신: 2026-10-27

## 한 줄 요약
음성 파일 업로드 기능에 mp4(영상)를 받아 `<audio>`로 재생하려면 응답 Content-Type을 `audio/mp4`로 주면 되지만, 허용 확장자 목록·전사 서버 지원 여부·용량 제한까지 함께 봐야 한다.

## 내용
### MIME 값
- m4a와 mp4는 같은 MP4 컨테이너다. m4a는 보통 `audio/mp4`로 내려보낸다.
- mp4 파일 자체의 형식은 `video/mp4`지만, `<audio>` 태그로 음성만 재생할 목적이면 `audio/mp4`가 의도에 맞다.
- MIME 매핑이 없어 `application/octet-stream`으로 내려가도 Chrome은 내용을 보고 재생하는 경우가 많지만, Safari는 이 값에 더 민감하다. 정확한 MIME을 주는 편이 안전하다.
- 탐색 재생(구간 이동)에는 Range 요청에 대한 `206 Partial Content` 응답이 필요하다.

### 브라우저 재생 조건
- 흔한 조합(H.264 영상 + AAC 음성)은 Chrome·Edge·Safari 모두 `<audio>`로 음성만 재생할 수 있다.
- 영상 트랙이 브라우저가 못 푸는 코덱(예: HEVC)이면 동작이 브라우저마다 다를 수 있으니 실제 파일로 확인한다.

### 기능 전체에서 확인할 것
- **허용 목록을 한 곳에서 관리**하고 화면 필터(`accept`), 안내 문구, 화면 검사, 서버 업로드 검사, 모바일 API가 모두 그 목록을 쓰게 하면 확장자 추가가 한 줄로 끝난다.
- **음성 인식(전사) 서버가 그 형식을 지원하는지**: 파일 경로만 넘기고 서버가 직접 읽는 구조면, 지원 여부와 "변환해 다른 파일(wav 등)로 저장하는지"를 서버 쪽에서 확인해야 한다. 변환해 저장한다면 화면의 파일명·경로도 바꿔 줘야 한다.
- **용량**: 영상은 크다. 업로드 크기 제한, 저장소 용량, 전사 시간 제한(긴 영상)을 함께 정한다.

## 핵심 코드

### 1) 확장자 → MIME
```java
static String mimeType(String ext) {
    switch (ext.toLowerCase()) {
        case "mp3":  return "audio/mpeg";
        case "m4a":
        case "mp4":  return "audio/mp4";
        case "wav":  return "audio/wav";
        case "aac":  return "audio/aac";
        case "webm": return "audio/webm";
        default:     return "application/octet-stream";
    }
}
```
허용 확장자 전부에 `audio/` MIME이 있는지 검사하는 단위 테스트를 두면, 나중에 확장자만 추가하고 MIME을 빠뜨리는 실수를 잡는다.

### 2) 허용 확장자 목록을 한 곳에 (Java 8)
```java
public final class MediaTypes {
    public static final List<String> ALLOWED =
            Collections.unmodifiableList(Arrays.asList("mp3", "m4a", "mp4", "wav", "aac", "webm"));

    public static String ext(String filename) {
        int dot = filename.lastIndexOf('.');
        return dot < 0 ? "" : filename.substring(dot + 1).toLowerCase(Locale.ROOT);
    }

    public static boolean allowed(String filename) { return ALLOWED.contains(ext(filename)); }   // 서버 업로드 검사

    public static String accept() {                       // 화면 <input accept> 값: ".mp3,.m4a,..."
        StringBuilder sb = new StringBuilder();
        for (String e : ALLOWED) sb.append(sb.length() == 0 ? "" : ",").append('.').append(e);
        return sb.toString();
    }
}
```
```html
<!-- 화면: 서버가 내려준 MediaTypes.accept() 값을 그대로 쓴다 -->
<input type="file" name="audio" accept=".mp3,.m4a,.mp4,.wav,.aac,.webm">
<audio controls preload="metadata" src="/media/play?id=123"></audio>
```

### 3) Range 요청에 206 으로 응답하는 재생 서블릿 (Java 8, 단일 구간만)
```java
protected void doGet(HttpServletRequest req, HttpServletResponse res) throws IOException {
    File f = resolveFile(req);                             // 직접 구현: 권한 확인 후 파일
    long len = f.length(), start = 0, end = len - 1;
    res.setHeader("Accept-Ranges", "bytes");
    res.setContentType(mimeType(MediaTypes.ext(f.getName())));
    String range = req.getHeader("Range");                 // 예: bytes=1000-  bytes=1000-1999  bytes=-500
    if (range != null && range.startsWith("bytes=")) {
        try {
            String[] r = range.substring(6).split("-", 2);
            if (r[0].isEmpty()) start = Math.max(0, len - Long.parseLong(r[1]));
            else { start = Long.parseLong(r[0]); if (!r[1].isEmpty()) end = Math.min(Long.parseLong(r[1]), len - 1); }
        } catch (RuntimeException e) { start = len; }       // 여러 구간·형식 오류 → 416
        if (start > end) { res.setHeader("Content-Range", "bytes */" + len); res.sendError(416); return; }
        res.setStatus(206);
        res.setHeader("Content-Range", "bytes " + start + "-" + end + "/" + len);
    }
    res.setHeader("Content-Length", String.valueOf(end - start + 1));
    try (RandomAccessFile raf = new RandomAccessFile(f, "r"); OutputStream out = res.getOutputStream()) {
        raf.seek(start);
        byte[] buf = new byte[8192];
        long left = end - start + 1;
        int n;
        while (left > 0 && (n = raf.read(buf, 0, (int) Math.min(buf.length, left))) > 0) { out.write(buf, 0, n); left -= n; }
    }
}
```
- 정적 파일을 톰캣 기본 서블릿으로 내보내면 Range는 이미 지원된다. 위 코드는 다운로드를 직접 구현할 때만 필요하다.

### 4) 확인 명령
```bash
# 응답 헤더만 보기: 206, Content-Type, Content-Range 확인
curl -s -o /dev/null -D - -H "Range: bytes=0-99" "https://<서버>/media/play?id=123"
# 실제 파일의 코덱 확인 (영상이 HEVC 인지 등)
ffprobe -v error -show_entries stream=codec_type,codec_name -of csv=p=0 sample.mp4
```

### 5) 허용 확장자 전부에 audio MIME 이 있는지 테스트 (JUnit 4)
```java
@Test
public void everyAllowedExtensionHasAudioMime() {
    for (String ext : MediaTypes.ALLOWED) {
        assertTrue(ext, mimeType(ext).startsWith("audio/"));   // 확장자만 추가하고 MIME 을 빠뜨리면 실패
    }
}
```

## 주의할 점
- 서버 응답 형식만 확인하려고 몇 바이트짜리 가짜 파일을 올리면 실제 재생 가능 여부는 확인되지 않는다. 진짜 파일로 따로 확인한다.
- 시험 업로드 파일은 "작성 취소" 정리 API로 지우고, 같은 경로가 404가 되는지로 삭제를 확인한다.

## 참고
- MDN: Media container formats, Web audio codec guide, HTTP 206 Partial Content
