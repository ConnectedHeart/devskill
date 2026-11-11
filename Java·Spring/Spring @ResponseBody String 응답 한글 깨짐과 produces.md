# Spring @ResponseBody String 응답 한글 깨짐과 produces

> 분류: Java·Spring · 최근 갱신: 2026-11-10

## 한 줄 요약
`@ResponseBody`로 String을 돌려주는데 `produces`도, 메시지 컨버터 설정도 없으면 Spring 기본값 ISO-8859-1로 인코딩돼 한글이 `?`로 바뀐다. 매핑에 `produces = "text/plain; charset=utf-8"`을 붙여 고친다.

## 내용
- String 반환값은 `StringHttpMessageConverter`가 쓴다. 이 컨버터의 기본 문자셋은 (Spring 5 기준) ISO-8859-1이다. ISO-8859-1에 없는 한글은 `?`로 바뀌어 나가며, 브라우저 쪽에서 디코딩을 바꿔도 되살릴 수 없다(이미 바이트가 `?`).
- `CharacterEncodingFilter`는 보통 **요청** 인코딩만 맞춘다. `forceEncoding`(또는 `forceResponseEncoding`)을 켜지 않았다면 응답에는 영향이 없다. "인코딩 필터가 있는데 왜 깨지지?"의 흔한 이유다.
- 같은 컨트롤러 안에서도 매핑마다 결과가 다를 수 있다. 한글 성공 메시지는 다른 경로(JSP·JSON)로 나가고, 실패 문구만 String으로 나가면 실패 때만 깨져 늦게 발견된다.
- 해결 방법
  1. **매핑별 `produces`** — 그 매핑만 바뀌므로 영향 범위가 작다. 응답 `Content-Type`이 `text/plain;charset=utf-8`이 된다.
  2. **전역 컨버터 설정** — `StringHttpMessageConverter`를 UTF-8로 등록. 앱의 모든 String 응답이 바뀌므로 기존 화면이 Content-Type이나 문자셋에 기대고 있지 않은지 확인해야 한다.
- `produces`를 붙이면 요청의 `Accept`와 맞지 않을 때 406이 날 수 있다. `fetch`나 XHR이 `Accept`를 따로 정하지 않으면(`*/*`) 문제없다.

## 핵심 코드
```java
// 매핑별 지정: 한글 문구를 String으로 돌려주는 곳에 붙인다
@RequestMapping(value = "/items/save.do", produces = "text/plain; charset=utf-8")
@ResponseBody
public String save(ItemForm form) {
    if (!form.isValid()) {
        return "FAIL:종료일시는 시작일시보다 늦어야 합니다.";
    }
    itemService.save(form);
    return "OK";
}
```

```xml
<!-- 전역 지정(영향 범위가 넓음): 모든 String 응답을 UTF-8로 -->
<mvc:annotation-driven>
  <mvc:message-converters>
    <bean class="org.springframework.http.converter.StringHttpMessageConverter">
      <constructor-arg value="UTF-8"/>
    </bean>
  </mvc:message-converters>
</mvc:annotation-driven>
```

```javascript
// 확인: 응답 헤더의 문자셋과 본문을 같이 본다
const res = await fetch('/items/save.do', { method: 'POST', body: formData });
console.log(res.headers.get('content-type')); // text/plain;charset=utf-8 이어야 함
console.log(await res.text());
```

## 주의할 점
- 화면이 응답 문자열을 비교(`startsWith('FAIL')` 등)한다면 영문 접두어는 깨지지 않으므로 기능은 동작하고 문구만 깨진다. 시나리오 검사에서 문구까지 비교해야 찾을 수 있다.
- 서버 쪽 검사를 고쳤다면 화면 검사를 거치지 않는 요청(직접 API 호출)으로 실패 응답을 받아 문구와 헤더를 함께 확인한다.
- 요청 쪽 깨짐(필터 URL 패턴)은 [서블릿 필터 URL 패턴과 한글 파일명 깨짐](서블릿 필터 URL 패턴과 한글 파일명 깨짐.md)을 본다.

## 참고
- Spring Framework 문서: `StringHttpMessageConverter`, `@RequestMapping(produces)`
