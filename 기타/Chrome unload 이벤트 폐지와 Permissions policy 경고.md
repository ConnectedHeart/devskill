# Chrome unload 이벤트 폐지와 Permissions policy 경고

> 분류: 기타 · 최근 갱신: 2026-10-20

## 한 줄 요약
콘솔의 `[Violation] Permissions policy violation: unload is not allowed in this document.`는 오류가 아니라 경고이며, 스크립트는 계속 돌지만 등록한 `unload` 핸들러는 실행되지 않는다.

## 내용
- Chrome은 페이지를 떠날 때 발생하는 `unload` 이벤트를 단계적으로 없애는 중이다. 뒤로·앞으로 캐시(bfcache)를 막고 모바일에서 신뢰성이 낮기 때문이다.
- 브라우저 기본값이나 사이트의 `Permissions-Policy` 헤더에서 `unload`가 허용되지 않은 문서에서 `window.onunload = …` 또는 `addEventListener("unload", …)`를 등록하면 이 경고가 찍힌다. 앞에 붙은 숫자는 같은 경고가 난 횟수다.
- 직접 쓰지 않아도 함께 불러오는 라이브러리(구버전 jQuery, 웹 에디터, 외부 iframe 등)가 내부에서 등록해 나올 수 있다. 출처는 개발자 도구에서 메시지를 펼쳐 파일명·줄 번호로 확인한다.

### 영향과 대응
- 다른 기능 오류의 원인이 되는 경우는 드물다. 다만 "창을 닫을 때 자식 팝업 닫기, 잠금 해제, 임시 데이터 정리"를 `unload`에서 하던 화면은 그 정리가 안 될 수 있다.
- 대체 이벤트: `pagehide`(페이지를 떠날 때), `visibilitychange`(탭이 숨겨질 때 상태 저장). 서버로 마지막 데이터를 보내야 하면 `navigator.sendBeacon`을 쓴다.

## 예시
```js
// unload 대신
window.addEventListener("pagehide", function (e) {
  // e.persisted 가 true면 bfcache에 들어간 것(나중에 돌아올 수 있음)
  cleanup();
});
```

## 주의할 점
- `beforeunload`는 "나가시겠습니까?" 확인용으로 아직 쓸 수 있지만 남용하면 bfcache를 막는다. 저장되지 않은 변경이 있을 때만 등록한다.
- 경고가 보인다고 원인으로 단정하지 말고, 실제 증상(정리가 안 됨)이 있을 때 해당 코드를 `pagehide`로 바꾼다.

## 참고
- Chrome for Developers: "Deprecating the unload event"
- MDN: Window: pagehide event, Permissions-Policy
