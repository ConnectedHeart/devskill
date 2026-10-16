# iframe src 변경과 브라우저 방문 기록

> 분류: 기타 · 최근 갱신: 2026-10-14

## 한 줄 요약
레이어 팝업용 iframe의 `src`를 바꿀 때마다 브라우저 세션 기록(history)이 하나씩 쌓여, 사용자가 뒤로가기를 여러 번 눌러야 이전 페이지로 돌아가는 문제가 생긴다. `contentWindow.location.replace()`로 바꾸면 기록이 남지 않는다.

## 내용
- iframe의 `src` 속성에 다른 주소를 넣거나 iframe 안에서 `location.href`로 이동하면 최상위 창의 joint session history에 항목이 추가된다. 뒤로가기는 이 항목부터 되돌리므로 화면은 그대로인 채 iframe만 이전 주소로 돌아간다.
- 흔한 원인: 페이지 로드 후 팝업을 "닫는" 함수가 iframe을 빈 페이지 주소로 바꾸는 것(초기값과 다른 주소면 +1), 팝업 열기·닫기마다 src 변경, 메뉴 새로고침을 `location.href`로 하는 것.
- `location.replace(url)`은 현재 기록 항목을 교체하므로 기록이 늘지 않는다. 다른 도메인 주소에도 동작한다. 새로고침 목적이면 `location.reload()`도 기록을 남기지 않는다.
- 공용 함수에 적용할 때는 "기록 남기지 않기" 옵션을 추가하고 `=== true`일 때만 replace를 쓰면, 기존 호출부(다른 값을 5번째 인자로 넘기던 곳 등)에 영향을 주지 않는다.
- 크롬 등은 사용자 상호작용(클릭) 없이 자동으로 생긴 기록 항목을 뒤로가기 때 건너뛰기도 한다(history manipulation intervention). 그래서 같은 코드에서도 팝업 안을 클릭했는지에 따라 필요한 뒤로가기 횟수가 달라 보일 수 있다.

## 예시
```javascript
// 기록이 쌓임
document.getElementById('layerFrame').src = '/blank.htm';
// 기록이 쌓이지 않음
document.getElementById('layerFrame').contentWindow.location.replace('/blank.htm');
```
원인 위치 찾기: 단계마다 콘솔에서 `history.length`를 찍어 어디서 늘어나는지 본다.

## 주의할 점
- `contentWindow.location.replace`를 쓰면 iframe의 `src` 속성값은 이전 주소로 남는다. 다른 코드가 `src` 속성을 읽어 현재 페이지를 판단한다면 어긋난다.
- 다른 출처(origin)의 iframe 안 페이지에서 일어나는 이동은 바깥에서 막을 수 없다.

## 참고
- HTML Living Standard — session history, Location.replace()
- MDN — History.length, Location.replace()
