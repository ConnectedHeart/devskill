# HTML select 펼친 목록에 CSS가 안 먹는 이유

> 분류: 기타 · 최근 갱신: 2026-10-19

## 한 줄 요약
`<select>`의 닫힌 상자는 CSS로 꾸밀 수 있지만, 펼쳤을 때 나오는 옵션 목록은 운영체제·브라우저가 직접 그리는 네이티브 UI라 대부분의 CSS가 적용되지 않는다. PC 브라우저로 모바일 화면을 볼 때 "이질적인" 목록이 보이는 이유다.

## 내용
- PC Chrome에서는 Windows 기본 목록(사각 테두리, 시스템 글꼴, 파란 선택 표시)이 뜬다. 실제 휴대폰에서는 iOS는 하단 휠 선택기, Android는 선택 대화상자로 바뀌어 아예 다른 모양이 된다.
- 모양을 통일하려면 네이티브 대신 라이브러리가 그리는 목록을 쓴다. 예: jQuery Mobile `data-native-menu="false"`(팝업 목록), 또는 커스텀 드롭다운 컴포넌트. 이 경우 접근성·모바일 조작감이 네이티브보다 떨어질 수 있다.
- 한 화면만 바꾸면 다른 화면과 오히려 달라진다. 같은 앱의 기존 select가 모두 네이티브라면 그대로 두는 것이 일관성 면에서 낫다.

### 같이 겪기 쉬운 모바일 팝업 글자 잘림
- 팝업 내용 영역에 `border-radius` + `overflow:hidden` + `padding:0`이 걸려 있으면, 문구가 한 줄을 꽉 채울 때 양 끝 글자가 둥근 모서리에 잘린다. 짧은 문구만 쓸 때는 드러나지 않는다.
- 문구 요소에 좌우 여백, `white-space:normal`, 한글은 `word-break:keep-all`을 주고, 팝업에 `max-width`를 두어 긴 문구가 여러 줄로 나뉘게 한다. 공통 규칙을 바꾸기보다 전용 클래스로 범위를 제한하면 다른 화면 영향이 없다.

## 예시
```html
<!-- 네이티브 (기본) -->
<select name="type"><option>전체</option><option>제목</option></select>
<!-- jQuery Mobile 자체 팝업 목록 -->
<select name="type" data-native-menu="false">...</select>
```
```css
.myDialog { max-width: 320px; }
.myDialog .title { padding: 0 18px; white-space: normal; word-break: keep-all; line-height: 150%; }
```

## 주의할 점
- 값을 스크립트로 바꾼 뒤 jQuery Mobile select는 `$(sel).selectmenu("refresh")`를 불러야 화면 글자가 갱신된다.

## 참고
- MDN: `<select>` 스타일링, `appearance`
- jQuery Mobile 문서: Select menu (nativeMenu)
