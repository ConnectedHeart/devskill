# 보기 화면에서 contenteditable 막기(속성 제거와 false)

> 분류: 기타 · 최근 갱신: 2026-11-11

## 한 줄 요약
저장된 HTML 본문에 `contenteditable` 속성이 섞여 들어가면 읽기 전용이어야 할 보기 화면에서 본문을 고칠 수 있다. 보기 화면에서 본문을 넣은 직후 속성을 **지우는** 것이 맞고, `false`로 바꾸면 그 화면이 본문을 다시 저장하는 경로에서 `false`가 파일에 남아 나중에 그 문서를 복사·재사용할 때 편집이 막힌다.

## 내용
- 입력 방식이 둘인 서식이 흔하다. ① 본문 자리에 리치 텍스트 편집기를 띄움 ② 편집기 없이 본문 DIV에 `contentEditable=true`를 붙여 직접 입력. ②에서 저장할 때 속성을 걷어 내지 않거나, 다른 경로(외부 작성 도구, API)로 만든 HTML에 속성이 들어 있으면 저장본에 남는다.
- 보기 화면이 `input`·`select`·`textarea`만 막고 `contenteditable`은 막지 않으면, 사용자가 본문을 고칠 수 있다. 화면 값이 다시 저장되는 기능(승인 시 화면 본문을 파일로 저장 등)이 있으면 원본 문서가 바뀌는 사고가 된다.
- 막는 위치: 저장 쪽만 고치면 이미 저장된 문서는 그대로다. 본문을 화면에 넣은 직후 보기 화면에서 막으면 과거 문서까지 한 번에 해결된다.
- 지우기 vs `false`
  - `false`: 그 화면에서 다시 저장되면 `contenteditable="false"`가 파일에 남는다. 그 문서를 템플릿처럼 복사해 새 문서를 쓰면, 안쪽 요소가 편집 불가로 남아 그 부분을 고칠 수 없다.
  - 속성 제거: 일반 DIV로 저장된다. 재사용 결과가 기준과 같다(실제 비교로 확인).
- 재사용 화면이 화면 본문이 아니라 서버 파일을 다시 읽는 구조라면, 보기 화면 수정은 "그 화면에서 다시 저장하는 경로"에서만 파일에 영향을 준다. 그 경로를 일부러 만들어 전후를 비교한다.
- 같은 본문 화면을 쓰는 다른 화면(미리보기, 팝업, 진행 중 문서 보기 등)에도 같은 처리가 필요한지 함께 정한다.
- 보기 화면이 고칠 수 있게 열려 있었다면, 저장 API(`/saveFile`류)가 서버에서 권한(문서 소유·결재 단계)을 확인하는지도 본다. 화면 막기는 보안이 아니다.

## 핵심 코드
```javascript
// 저장된 본문을 보기 영역에 넣은 직후 호출
function makeReadOnly(container) {
  container.querySelectorAll("input, select, textarea").forEach(function (el) { el.disabled = true; });
  // false로 바꾸지 말고 속성을 지운다 (다시 저장될 때 흔적이 남지 않게)
  container.querySelectorAll("[contenteditable]").forEach(function (el) {
    el.removeAttribute("contenteditable");
  });
}
```
```javascript
// 회귀 확인 예: 보기 → (다시 저장) → 재사용 편집 화면에서 편집 가능 요소 수 비교
const editableCount = await frame.evaluate(() =>
  document.querySelectorAll('[contenteditable="true"], [contenteditable=""]').length);
const lockedCount = await frame.evaluate(() =>
  document.querySelectorAll('[contenteditable="false"]').length);   // 0이어야 함
```

## 주의할 점
- 원인이 "저장 때 남음"인지 "다른 경로로 들어옴"인지는 문제 문서 파일을 직접 봐야 확정된다. 재현 데이터로 확인되지 않은 원인을 단정하지 않는다.
- 문서를 다른 채널(메일 본문, 게시판 게시, 모바일 보기)로 옮기는 기능이 있으면 그쪽에서도 편집 가능한지 확인한다.

## 참고
- MDN: contenteditable, HTMLElement.isContentEditable
- 관련 노트: [화면 제한과 서버 측 접근 확인(IDOR)](../인증·보안/화면 제한과 서버 측 접근 확인(IDOR).md)
