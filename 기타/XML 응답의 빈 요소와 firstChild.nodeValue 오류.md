# XML 응답의 빈 요소와 firstChild.nodeValue 오류

> 분류: 기타 · 최근 갱신: 2026-11-12

## 한 줄 요약
DOM으로 XML을 읽을 때 `<NAME/>`처럼 빈 요소에는 자식 텍스트 노드가 없어 `el.firstChild`가 `null`이다. 서버가 빈 값을 `<![CDATA[]]>`로 채워 주던 경로가 하나라도 빠지면 `Cannot read properties of null (reading 'nodeValue')`로 화면 그리기가 통째로 멈춘다.

## 내용
- `<NAME>abc</NAME>` → 텍스트 노드가 하나 있어 `firstChild.nodeValue === "abc"`.
- `<NAME/>`, `<NAME></NAME>` → 자식이 없어 `firstChild === null`.
- `<NAME><![CDATA[]]></NAME>` → 브라우저에 따라 빈 CDATA 노드가 생겨 `firstChild`가 있다. 그래서 서버가 빈 값을 늘 CDATA로 감싸 보내면 클라이언트의 `firstChild.nodeValue` 코드가 우연히 돌아간다.
- **흔한 깨짐 시나리오**: 성능 개선으로 "하위 노드를 처음부터 중첩해서 한 번에 내려보내기" 같은 새 응답 경로를 추가했는데, 빈 값 채우기는 기존 경로(최상위 노드, 따로 받는 하위 응답)에만 걸려 있다. 새로 중첩된 노드만 빈 요소로 나가고, 트리 컴포넌트가 첫 빈 요소에서 예외로 멈춰 메뉴·목록이 비고 이어지는 동작(첫 항목 자동 선택 등)도 실행되지 않는다.
- 빈 값이 흔한 컬럼(URL, 설명)일수록 실제 데이터에서 바로 터진다. 시험 데이터가 모두 채워져 있으면 개발 때는 안 보인다.

### 원인 좁히기
1. 콘솔 오류의 파일·행에서 `firstChild.nodeValue`를 찾는다.
2. 네트워크 탭에서 응답 XML을 열어 빈 요소(`<X/>`)가 있는지 세어 본다.
3. 새 기능을 끄는 요청 값으로 바꿔(예: 중첩 응답을 안 받게) 정상이면 새 경로가 원인이다.
4. 클라이언트 한 줄만 null을 견디게 바꿔 끼워 정상이 되는지 확인한다.

## 핵심 코드
### 1) 클라이언트: null을 견디게 읽기
```javascript
function textOf(node) {
  // 빈 요소는 firstChild가 null, textContent는 ""를 돌려준다
  return node && node.firstChild ? node.firstChild.nodeValue : "";
}
// 또는 자식 텍스트를 모두 합쳐 읽기
var value = node.textContent || "";
```

### 2) 서버: 빈 값 채우기를 모든 경로에서 같은 함수로
```java
// 응답 XML을 만드는 곳이 여러 개면 빈 값 처리를 한 함수로 모은다
static String cdata(String v) {
    return "<![CDATA[" + (v == null ? "" : v.replace("]]>", "]]]]><![CDATA[>")) + "]]>";
}
// 최상위 노드, 별도 하위 응답, 새로 추가한 중첩 노드 모두 이 함수를 거치게 한다
sb.append("<URL>").append(cdata(item.getUrl())).append("</URL>");
```

### 3) 응답에서 빈 요소 찾기 (브라우저 콘솔)
```javascript
const xml = new DOMParser().parseFromString(responseText, "text/xml");
const empties = [...xml.getElementsByTagName("*")].filter(e => !e.firstChild && e.children.length === 0);
console.log(empties.length, empties.slice(0, 10).map(e => e.tagName));
```

## 주의할 점
- 클라이언트만 고치면 같은 응답을 쓰는 다른 화면이 여전히 깨질 수 있다. 서버 쪽 빈 값 처리를 모든 경로에 맞추는 것이 근본 수정이다.
- CDATA 안에 `]]>`가 들어가면 문서가 깨진다. 값에 들어갈 수 있으면 나눠서 감싼다.
- 시험 데이터에 빈 값 행을 일부러 넣어 둔다.

## 참고
- MDN — Node.firstChild, Node.nodeValue, Node.textContent
