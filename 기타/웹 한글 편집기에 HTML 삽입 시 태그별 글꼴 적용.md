# 웹 한글 편집기에 HTML 삽입 시 태그별 글꼴 적용

> 분류: 기타 · 최근 갱신: 2026-11-19

## 한 줄 요약
웹 기반 한글(HWP) 편집기에 HTML을 넣으면 `p`·`div`·`span`에 준 글꼴 style만 반영되고, `li`·`td`·`th`·`h1~h6`·`pre` 바로 안의 글자는 style을 줘도 HTML 기본 글꼴(굴림 10pt 등)로 들어간다. 글자를 `span`이나 `p`로 감싸서 넘기면 본문 글꼴로 맞출 수 있다.

## 내용
- AI 답변처럼 외부에서 받은 HTML을 문서 편집기의 본문 필드에 "HTML 형식으로 붙이기"로 넣으면, 편집기가 HTML을 자기 문단·글자 모양으로 변환한다. 이때 어떤 태그의 style을 읽는지는 편집기 변환기 구현에 달려 있다.
- 실측 결과(본문이 돋움 12pt인 양식, 누름틀·표 칸 양식 모두 같음):

| 넣은 모양 | 들어간 글꼴 |
|---|---|
| `<p style="본문글꼴">`, 그 안의 `strong`·`em`·`a`·`code` | 본문 글꼴 ✓ |
| `<div>`, `<blockquote><p>` | 본문 글꼴 ✓ |
| `<li>글자</li>`, `<li><strong>굵게</strong> 글자</li>` | 굴림 10pt ✗ (굵게는 유지) |
| `<li><p>글자</p></li>`, `<li><span style>…</span></li>` | 본문 글꼴 ✓ |
| `<h1>`~`<h6>` (안에 strong이 있어도) | 굴림 24·18·14·12·8pt ✗ |
| `<td>글자</td>`, `<th>` | 굴림 10pt ✗ |
| `<td><p>글자</p></td>` | 본문 글꼴 ✓ |
| `<pre><code>`, 블록 없는 맨 글자 | 굴림 ✗ |

- `li`·`td`·`h`에 style을 더 많이 붙이는 방식으로는 고쳐지지 않는다. 변환기가 그 태그의 style을 통째로 무시하기 때문이다. `strong` 자체의 style도 무시되고, 감싸는 `p`·`span`의 글꼴을 물려받는다.
- 바깥 `div` 하나에 style을 주는 방법도 반영되지 않았다. 글자를 직접 감싸는 요소에 줘야 한다.
- 같은 목록이라도 생성 쪽이 `li`만 보내느냐 `li>p`로 보내느냐에 따라 결과가 달라지므로, 넣기 직전에 받은 쪽에서 모양을 정규화하는 편이 결과가 일정하다.
- 본문 글꼴을 "본문 필드 끝 글자"에서 읽으면, 표 칸처럼 끝에 이전 삽입 글이 남는 양식에서 두 번째 삽입 때 앞서 넣은 글(굴림)의 모양을 읽는다. 필드의 처음 글자에서 읽는 것이 안전하다.
- `li>p`는 일부 편집기에서 빈 글머리표 줄을 만들 수 있어 `li>span`이 더 안전했다.

## 핵심 코드
```javascript
// 편집기에 넘기기 전에 글꼴이 안 먹는 태그 안 글자를 span/p로 감싼다. bodyFont 예: "font-family:돋움;font-size:12pt"
function normalizeForHwp(container, bodyFont) {
  container.querySelectorAll('li').forEach(function (li) {
    var span = document.createElement('span');
    span.setAttribute('style', bodyFont);
    while (li.firstChild) span.appendChild(li.firstChild);   // 중첩 ul/ol은 따로 처리
    li.appendChild(span);
  });
  container.querySelectorAll('td, th').forEach(function (cell) {
    var p = document.createElement('p');
    p.setAttribute('style', bodyFont);
    while (cell.firstChild) p.appendChild(cell.firstChild);
    cell.appendChild(p);
  });
  container.querySelectorAll('h1,h2,h3,h4,h5,h6').forEach(function (h) {
    var p = document.createElement('p');                      // 제목은 굵게·조금 크게
    p.setAttribute('style', bodyFont + ';font-weight:bold');
    while (h.firstChild) p.appendChild(h.firstChild);
    h.parentNode.replaceChild(p, h);
  });
  return container.innerHTML;
}
```

## 주의할 점
- 생성 쪽 HTML에 인라인 `font-family` style이 들어 있으면 HTML 살균기(DOMPurify 기본 설정)는 인라인 style을 지우지 않으므로 그 값이 이길 수 있다.
- 제목을 `p`로 바꾸면 제목 크기는 직접 정해야 한다(본문 +2~4pt 등).
- 목록 기호와 들여쓰기는 편집기 변환 방식을 따르므로 양식의 목록 문단 모양과 똑같이 맞추기 어렵다.
- 한글 글꼴 이름만 넘기면 영문·숫자 글꼴이 다른 양식에서는 차이가 남을 수 있다.
- 편집기 버전마다 결과가 다를 수 있으니 태그별 시험 표를 만들어 직접 확인한다.
