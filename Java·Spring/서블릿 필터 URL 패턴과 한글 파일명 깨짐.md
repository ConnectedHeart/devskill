# 서블릿 필터 URL 패턴과 한글 파일명 깨짐

> 분류: Java·Spring · 최근 갱신: 2026-10-19

## 한 줄 요약
문자 인코딩 필터가 `*.do` 같은 확장자 패턴에만 매핑돼 있으면, 확장자 없는 REST 스타일 주소로 들어온 multipart 업로드는 필터를 타지 않아 한글 파일명과 폼 값이 깨진다. 주소를 필터 패턴에 맞추거나 필터 매핑을 넓혀야 한다.

## 내용
- 요청 본문(폼 파라미터, multipart 파트 헤더의 파일명)의 문자셋은 `request.setCharacterEncoding()`이 **파라미터를 처음 읽기 전에** 호출돼야 적용된다. 보통 Spring의 `CharacterEncodingFilter`가 이 일을 한다.
- 필터는 `web.xml`의 `<filter-mapping><url-pattern>`에 맞는 요청에만 걸린다. `*.do`로만 매핑하면 `/api/users/{id}/upload`처럼 확장자 없는 주소는 필터를 건너뛰고, 컨테이너 기본값(ISO-8859-1 등)으로 해석돼 한글이 `ê°€...`처럼 깨진다.
- 같은 애플리케이션 안에서 오래된 화면은 `.do`, 새로 만든 API는 확장자 없는 주소일 때 자주 생긴다. GET 쿼리스트링은 커넥터의 `URIEncoding`이 따로 결정하므로 GET은 멀쩡하고 POST·업로드만 깨지는 식으로 나타난다.
- 해결 방법
  1. 새 주소를 기존 필터 패턴에 맞춘다(예: `.../upload.do`).
  2. 필터 매핑에 `/*`나 해당 경로 패턴을 추가한다. 다른 필터(XSS 필터 등)도 같은 패턴을 쓰는지, 범위를 넓혔을 때 영향이 있는지 함께 본다.
- **경로 변수와 확장자**: Spring MVC에서 `{userId:.+}`처럼 점을 포함하도록 만든 경로 변수를 주소 맨 끝에 두고 `.do`를 붙이면, 정규식이 `.do`까지 변수 값으로 가져간다. 확장자를 붙일 때는 경로 변수를 앞쪽으로 옮기고 마지막 조각을 고정 문자열(`/list.do`)로 둔다.
- 주소를 바꾸면 인터셉터·보안 설정의 경로 패턴(`/api/**`, `/app*/**` 등)에 여전히 맞는지도 확인한다.

## 예시
```xml
<filter-mapping>
  <filter-name>encodingFilter</filter-name>
  <url-pattern>*.do</url-pattern>   <!-- 이 패턴 밖의 요청은 인코딩 설정이 안 됨 -->
</filter-mapping>
```
```java
// 나쁨: {userId:.+}.do 형태가 되면 ".do"까지 userId로 들어감
@RequestMapping("/records/users/{userId:.+}")
// 좋음: 변수를 앞으로, 마지막은 고정 이름 + 확장자
@RequestMapping("/users/{userId}/records/{recordId}/view.do")
```

## 주의할 점
- 필터 순서도 중요하다. 인코딩 필터보다 먼저 파라미터를 읽는 필터가 있으면 인코딩 설정이 무시된다.
- 호출하는 쪽(다른 서버의 HTTP 클라이언트)도 같은 변경에서 주소를 함께 바꾸고, 매핑 목록과 호출 목록을 대조한다.

## 참고
- Servlet 명세: filter-mapping url-pattern 규칙
- Spring Framework 문서: CharacterEncodingFilter, URI patterns
