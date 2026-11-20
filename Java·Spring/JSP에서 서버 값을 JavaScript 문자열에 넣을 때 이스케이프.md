# JSP에서 서버 값을 JavaScript 문자열에 넣을 때 이스케이프

> 분류: Java·Spring · 최근 갱신: 2026-11-19

## 한 줄 요약
JSP에서 `var title = "<c:out value='${title}'/>";`처럼 HTML 이스케이프 값을 JS 문자열에 넣으면 `&`·따옴표가 `&amp;`·`&#039;` 글자 그대로 JS 값이 된다. JS 문자열에는 JavaScript 이스케이프(`spring:escapeBody javaScriptEscape`)를 써야 한다.

## 내용
- `c:out`(또는 `fn:escapeXml`)은 HTML 본문에 넣을 때를 위한 것이다. 브라우저가 HTML로 해석할 때만 `&amp;`가 `&`로 돌아온다. `<script>` 안 JS 문자열 리터럴은 HTML 엔티티를 풀지 않으므로 `&amp;` 다섯 글자가 그대로 값이 된다.
- 이 값이 화면이 아니라 다른 곳(API 호출, 위젯에 보내는 컨텍스트, 파일명)으로 가면 "A&amp;B" 같은 제목이 그대로 전달된다.
- 반대로 이스케이프 없이 `"${title}"`로 넣으면 따옴표나 줄바꿈이 든 값에서 스크립트 문법 오류가 나고, `</script>`가 든 값은 스크립트 블록을 끊어 XSS가 된다.
- JS 문자열용으로는 따옴표·역슬래시·줄바꿈·`/`를 `\'`, `\\`, `\n`, `\/` 식으로 바꾸는 JavaScript 이스케이프를 쓴다. Spring 태그 `spring:escapeBody`에 `javaScriptEscape="true"`, `htmlEscape="false"`를 주면 된다.
- 같은 변수를 HTML 출력에도 쓰고 있다면 용도별로 변수를 나눈다(하나는 HTML 이스케이프, 하나는 JS 이스케이프).
- 모델에 넣지 않은 이름(`${title}`)을 쓰면 오류 없이 빈 값이 된다. 값이 늘 비어 있으면 모델 속성 이름부터 확인한다.

## 핵심 코드
```jsp
<%@ taglib prefix="spring" uri="http://www.springframework.org/tags" %>
<script>
  // 잘못된 예: '&'가 '&amp;'로 JS 값에 남는다
  var badTitle = "<c:out value='${item.title}'/>";

  // JS 문자열 리터럴용 이스케이프
  var title = "<spring:escapeBody htmlEscape='false' javaScriptEscape='true'>${item.title}</spring:escapeBody>";
  sendContext({ title: title });
</script>
```

```java
// 서버에서 JSON으로 넘기는 방법도 있다(Jackson). 문자열을 그대로 직렬화해 JS 객체로 쓴다.
model.addAttribute("itemJson", objectMapper.writeValueAsString(item));
// JSP: var item = ${itemJson};  ← 이때도 "</script>" 대비로 '<'를 <로 바꾸는 설정을 둔다
```

## 주의할 점
- `javaScriptEscape`만 쓰고 HTML 속성(`onclick="..."`) 안에 넣는 경우는 HTML 이스케이프도 함께 필요하다. 넣는 위치(HTML 본문, 속성, JS 문자열, URL)마다 이스케이프 종류가 다르다.
- 파일명처럼 다른 용도로 쓰는 값은 줄바꿈·따옴표를 따로 제거한다.
