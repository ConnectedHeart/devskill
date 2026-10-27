# 외부 응답을 화면에 넣는 방식(텍스트 노드와 innerHTML)

> 분류: 기타 · 최근 갱신: 2026-10-27

## 한 줄 요약
AI 서버 같은 외부 응답을 화면에 넣을 때 텍스트 노드로 넣으면 안전하지만 HTML 태그가 글자로 보이고, `innerHTML`로 넣으면 모양은 살지만 XSS와 원치 않는 컨트롤 노출이 따라온다. 어느 쪽으로 할지와 응답 형식을 함께 정해야 한다.

## 내용
### 텍스트로 넣을 때
- `document.createTextNode(value)`나 `textContent`로 넣으면 값이 HTML로 해석되지 않는다. 응답에 스크립트가 섞여 있어도 실행되지 않는다.
- 대신 응답이 `<ul><li>…</li></ul>` 같은 HTML 문자열이면 태그가 그대로 글자로 보인다. LLM은 목록 형태의 내용을 HTML이나 마크다운으로 포맷해서 답하는 경우가 많다.
- 해결 방향:
  1. 응답을 만드는 쪽에서 일반 텍스트로 답하게 한다(목록은 줄바꿈과 `- ` 기호). 화면은 줄바꿈만 `<br>`로 바꾸면 된다. 가장 단순하다.
  2. 화면에서 태그를 텍스트로 푼다. `<li>` → 줄바꿈 + `- `, `<br>`·`</p>` → 줄바꿈, 나머지 태그는 제거. 응답 형식이 들쭉날쭉할 때의 방어책.
  3. 제한된 태그만 허용해 HTML로 렌더링한다(`ul/ol/li/br/p/b` 등 화이트리스트). 모양은 가장 좋지만 정제 로직이 필요하다.

### HTML로 넣을 때
- `innerHTML`은 문자열 안의 모든 요소를 그대로 만든다. 다른 화면의 본문 HTML을 통째로 가져와 미리보기 영역에 넣으면, 그 안에 있던 버튼·입력 컨트롤(행 추가/삭제 버튼 등)까지 그려진다. "어디서 나온 버튼인지 모르겠다"는 증상의 원인이 된다.
- 표시용으로만 쓸 때는 사본에서 `input`, `button` 등을 제거하거나 CSS로 숨긴다. 같은 HTML을 나중에 원래 자리에 되돌려 넣는 기능이 있으면 원본에서 지우면 안 되고 표시용 사본에서만 뺀다.

### 서버 쪽 필터와 사용자 정의 속성
- 저장 경로의 필터가 "위험한 태그만 제거"(script, object 등) 방식이면 `data-*` 같은 사용자 정의 속성은 남는다. "허용 목록(화이트리스트)" 방식이면 목록에 없는 속성은 지워진다.
- 같은 본문이라도 경로마다 필터가 다를 수 있다. 편집 화면으로 불러올 때는 그대로 오고, 조회 화면 응답에서는 화이트리스트로 속성이 빠지는 식이다. `data-*` 속성에 기능을 걸어 둘 때는 저장 → 편집 재로드 → 조회 각 경로에서 속성이 살아남는지 따로 확인한다.
- 에디터 자체의 속성 필터도 확인한다. CKEditor는 `allowedContent` 설정에 따라 허용하지 않은 속성을 제거한다.

## 핵심 코드

### 1) 텍스트로 넣기와 목록 태그 풀기
```javascript
// 안전하게 넣기: 줄바꿈만 <br>로
function setText(el, value) {
    el.textContent = "";
    String(value).split(/\r\n|\r|\n/).forEach(function (line, i) {
        if (i > 0) el.appendChild(document.createElement("br"));
        el.appendChild(document.createTextNode(line));
    });
}

// 목록 태그를 텍스트로 풀기
function htmlToPlain(s) {
    return s.replace(/<li[^>]*>/gi, "\n- ")
            .replace(/<br\s*\/?>|<\/p>/gi, "\n")
            .replace(/<[^>]+>/g, "")
            .trim();
}
```

### 2) 허용 태그만 HTML로 렌더링 (DOMPurify)
```html
<script src="https://cdn.jsdelivr.net/npm/dompurify@3.1.6/dist/purify.min.js"></script>
```
```javascript
function setLimitedHtml(el, value) {
    el.innerHTML = DOMPurify.sanitize(String(value), {
        ALLOWED_TAGS: ["ul", "ol", "li", "br", "p", "b", "strong", "em"],   // 화이트리스트
        ALLOWED_ATTR: []                                                    // 속성은 모두 제거
    });
}
```

### 3) 다른 화면의 HTML을 미리보기로 넣을 때 컨트롤 빼기
```javascript
function showPreview(target, html) {
    // DOMParser 로 만든 문서는 스크립트·이벤트가 실행되지 않는다. 원본 html 문자열은 그대로 둔다
    var doc = new DOMParser().parseFromString(html, "text/html");
    doc.body.querySelectorAll("input, button, select, textarea").forEach(function (n) { n.remove(); });
    target.innerHTML = doc.body.innerHTML;     // 신뢰할 수 없는 값이면 여기서 DOMPurify 를 거친다
}
```
```css
/* 또는 지우지 않고 미리보기 영역에서만 숨기기 */
.preview input, .preview button, .preview select, .preview textarea { display: none; }
```

### 4) 서버 쪽 화이트리스트 필터 (jsoup, Java 8)
```java
// jsoup 1.14.1+ 는 Safelist (이전 버전은 Whitelist). 목록에 없는 태그·속성은 지워진다
Safelist list = Safelist.basic()
        .addAttributes(":all", "data-ref");          // data-* 를 쓰려면 이름을 하나씩 등록해야 남는다
String safe = Jsoup.clean(html, list);
```

### 5) CKEditor 4: 사용자 정의 속성 지우지 않게 하기
```javascript
CKEDITOR.replace("editor", {
    extraAllowedContent: "span[data-*]; div[data-*]"   // 허용 콘텐츠 필터(ACF)에 추가
});
```

## 주의할 점
- 줄바꿈으로 나눌 때 `split("\r|\n")`처럼 쓰면 `\r\n`이 두 번 나뉘어 빈 줄이 하나 더 생긴다. `\r\n`을 먼저 매칭한다(`\r\n|\r|\n`).
- 정규식으로 태그를 걷어내는 것은 표시용 정리일 뿐 보안 필터가 아니다. HTML로 렌더링할 값은 검증된 정제 라이브러리나 화이트리스트를 거친다.
- 조건을 만족하지 못해 기능 버튼이 조용히 안 나타나는 화면은 원인 찾기가 어렵다. 서버 조건(설정, 모드)과 클라이언트 조건(로딩 완료, 대상 요소 존재)을 나눠 순서대로 확인한다.

## 참고
- MDN: Node.textContent, Element.innerHTML, Document.createTextNode
- OWASP Cross Site Scripting Prevention Cheat Sheet
