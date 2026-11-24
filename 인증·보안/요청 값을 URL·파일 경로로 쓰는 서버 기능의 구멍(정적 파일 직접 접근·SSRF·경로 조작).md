# 요청 값을 URL·파일 경로로 쓰는 서버 기능의 구멍(정적 파일 직접 접근·SSRF·경로 조작)

> 분류: 인증·보안 · 최근 갱신: 2026-11-23

## 한 줄 요약
문서 ID 권한 검사를 아무리 고쳐도, 업로드 파일이 로그인 없이 정적 파일로 내려가거나, 서버가 요청에 든 URL을 그대로 열거나, 요청에 든 파일명을 그대로 경로에 붙이면 권한 검사를 통째로 건너뛰는 길이 생긴다.

## 내용
### ID를 숨기는 것은 대책이 아니다
- 브라우저가 문서를 요청하려면 어떤 식별자든 보내야 하고, 브라우저에 보낸 값은 개발자도구에서 모두 보인다. 연도+일련번호 같은 ID는 숨겨도 추측된다. F12 막기·난독화는 효과가 없다.
- 대책은 요청마다 서버가 DB 기준으로 "이 사용자가 이 대상에 접근할 수 있는가"를 판정하는 것이다([화면 제한과 서버 측 접근 확인(IDOR)](화면 제한과 서버 측 접근 확인(IDOR).md)).
- 판정을 건너뛰게 하는 요청 값(`view=manager`, `shared=true` 같은 플래그)이 있으면, 그 값을 붙이는 것만으로 남의 문서가 열린다. 플래그를 받더라도 서버가 그 사용자의 관리자 권한·공유 기록을 따로 확인해야 한다.
- 부서·회사처럼 판정에 쓰는 사용자 속성을 쿠키에서 읽으면 쿠키를 바꿔 다른 부서로 판정받을 수 있다. 서버 세션이나 DB에서 읽는다.

### 1) 업로드 저장 폴더가 정적 파일로 내려가는 경우
- Spring MVC의 `<mvc:default-servlet-handler/>`는 매핑되지 않은 요청을 WAS 기본 서블릿으로 넘겨 웹 루트 아래 파일을 그대로 내려준다. 업로드 폴더가 웹 루트 아래(또는 그쪽으로 링크)에 있고 로그인 인터셉터가 그 경로를 제외하면, 쿠키 없이 파일이 200으로 받아진다.
- 파일 경로가 ID로 정해지는 구조(`/files/<연도>/<번호>/<ID>.확장자`)라면 ID만 알면 다른 모든 검사를 우회한다. 쿠키 없는 `curl`로 바로 확인할 수 있다.
- 해결: 업로드 폴더를 웹 루트 밖으로 두고 권한 검사를 거치는 다운로드 컨트롤러로만 내보내거나, 그 경로를 인증 인터셉터·보안 필터 대상에 넣는다. 앞단 웹서버가 같은 경로를 직접 서비스하지 않는지도 본다.

### 2) 서버가 요청에 든 URL을 여는 경우(SSRF)
- `new URL(요청값).openStream()`처럼 서버가 받은 URL을 그대로 열면, `file:///`로 서버 디스크의 아무 파일이나, `http://내부주소`로 내부망 자원을 읽어 응답(zip 등)에 담아 갈 수 있다.
- 해결: URL을 클라이언트에서 받지 말고 서버가 ID로 만든다. 꼭 받아야 하면 스킴을 `https`로, 호스트를 허용 목록으로 제한하고, 이름 해석 뒤 사설·루프백 주소를 거부한다.

### 3) 요청에 든 파일명을 경로에 붙이는 경우(경로 조작)
- `Paths.get(폴더 + 파일명)`에 `..\..\` 같은 이름이 오면 의도한 폴더 밖에 쓰거나 읽는다. 끝에 정리(삭제) 코드가 경로 검사에서 예외를 내면 파일이 남는다.
- 해결: 파일명에서 경로 구분자를 지우거나 서버가 이름을 정하고, `normalize()` 뒤 결과가 기준 폴더 아래인지 `startsWith`로 확인한다.

### 4) "검사 대상 밖 경로는 통과" 규칙
- 다운로드 권한 검사가 특정 업무 폴더 경로만 판정하고 나머지는 "기존 동작 유지"로 통과시키면, 클라이언트가 보낸 경로 목록에 업무 폴더 밖 경로를 넣어 검사를 피할 수 있다. 클라이언트가 보낸 경로 목록을 쓰지 말고 서버가 ID로 경로를 다시 만든다.

## 핵심 코드
```bash
# 1) 업로드 파일이 로그인 없이 받아지는지 (쿠키 없이 요청)
curl -s -o /dev/null -w "%{http_code} %{size_download}\n" \
  "http://localhost:8080/files/2026/0001/12345.docx"   # 200이면 우회 가능
```

```java
// 2) 받은 URL을 열기 전에 스킴·호스트를 제한한다
URI uri = URI.create(urlParam);
if (!"https".equals(uri.getScheme()) || !ALLOWED_HOSTS.contains(uri.getHost())) {
    throw new IllegalArgumentException("not allowed");
}
InetAddress addr = InetAddress.getByName(uri.getHost());
if (addr.isLoopbackAddress() || addr.isSiteLocalAddress() || addr.isLinkLocalAddress()) {
    throw new IllegalArgumentException("internal address");
}
```

```java
// 3) 파일명은 기준 폴더 안으로만
Path base = Paths.get(saveDir).toAbsolutePath().normalize();
Path target = base.resolve(Paths.get(fileName).getFileName().toString()).normalize();
if (!target.startsWith(base)) throw new IllegalArgumentException("bad path");
```

```xml
<!-- 1) 업로드 경로도 인증 인터셉터를 거치게 (제외 목록에 넣지 않는다) -->
<mvc:interceptors>
  <mvc:interceptor>
    <mvc:mapping path="/**"/>
    <mvc:exclude-mapping path="/css/**"/>
    <mvc:exclude-mapping path="/js/**"/>
    <bean class="com.example.web.LoginCheckInterceptor"/>
  </mvc:interceptor>
</mvc:interceptors>
```

## 주의할 점
- 인터셉터는 기본 서블릿으로 넘어가는 요청에 걸리지 않는 설정도 있다. 쿠키 없는 요청으로 실제 응답 코드를 꼭 확인한다.
- 오브젝트 스토리지 모드나 앞단 웹서버 설정에 따라 운영 결과가 로컬과 다를 수 있으니 환경마다 확인한다.
- 문서 ID를 받는 API는 수가 많다. 조회만이 아니라 변경·삭제·묶음 저장 API까지 전수 목록을 만들어 판정 호출 유무를 본다.

## 참고
- OWASP Top 10 — A01 Broken Access Control, A10 Server-Side Request Forgery
- OWASP Path Traversal
- Spring Framework 문서 — `mvc:default-servlet-handler`
