# window.open 창 이름 재사용과 opener

> 분류: 기타 · 최근 갱신: 2026-10-30

## 한 줄 요약
`window.open(url, name)`은 같은 이름의 창이 이미 있으면 새 창을 만들지 않고 그 창을 새 주소로 다시 로드한다. 이 때문에 작성 중인 창의 내용이 사라지거나, 고정 이름 보조 창의 `opener`가 예전 창에 남아 엉뚱한 문서가 처리될 수 있다. 다른 출처(위젯, 새 탭 링크)에서 연 창에는 `opener`가 없어 본창에 기대는 기능이 깨진다.

## 내용
### 같은 이름 창은 재사용된다
- 창 이름을 문서 ID로 쓰면 "같은 문서는 한 창"이 되어 편하다. 하지만 같은 문서를 다른 화면(보기·결재·편집)으로 여는 경로가 여럿이면, 편집 중인 창이 보기 화면으로 다시 로드되면서 입력 내용이 사라진다.
- 경로마다 이름을 다르게 준다(`view_<id>`, `edit_<id>`). 또는 이미 열린 창이 있으면 다시 로드하지 않고 `focus()`만 한다.

### 고정 이름 보조 창과 opener
- 양식 선택처럼 이름이 고정된 보조 창(`"formSelect"`)을 문서 A 창에서 띄운 채, 문서 B 창에서 같은 보조 창을 다시 열면 기존 창이 다시 로드된다.
- 브라우저는 이미 있는 이름의 창을 재사용할 때 `opener`를 바꾸지 않을 수 있다. 그러면 보조 창의 [확인]이 A 창의 콜백을 불러 A가 처리된다. 보조 창 이름에 부모를 구분하는 값(창마다 만든 UUID 등)을 붙이고, 콜백 대상을 `opener`가 아니라 명시적으로 넘긴다.
- 연 창을 전역 변수 하나에 덮어쓰면 "열린 창 닫기"가 마지막 창만 닫는다. 목록(배열·Map)으로 관리한다.

### 다른 곳에서 연 창에는 opener가 없다
- 새 탭 링크(`target=_blank`, 기본 `noopener`)나 다른 출처의 위젯이 연 창에서는 `window.opener`가 null이다. 본창 함수를 부르는 버튼이 예외로 멈춘다.
- 해결 방향: 위젯은 "이 문서를 열어 달라"는 메시지(`postMessage`)만 보내고, 창은 본창(호스트)이 기존 함수로 연다. 그러면 평소 경로와 같은 창이 된다. 메시지로 받은 ID는 호스트가 다시 검사하고, 권한은 기존 보기 화면이 판정한다.

### 팝업 차단
- 팝업 허용은 사용자 클릭 직후의 동기 호출에만 주어진다. 클릭 → ajax → 콜백에서 `window.open`을 부르면 차단될 수 있다. 차단되면 반환값이 null이라 이어지는 `.focus()`가 예외를 낸다.
- 비동기로 값을 받아야 하면 클릭 즉시 빈 창을 열어 두고, 응답이 오면 그 창의 `location`을 바꾼다. 또는 필요한 값을 서버가 조회하게 해서 클릭 즉시 최종 주소로 연다.
- 팝업 허용은 출처(호스트·포트)별이다. 같은 서비스라도 다른 주소로 접속한 탭에서만 막힐 수 있다.

## 핵심 코드
### 1) 경로별 창 이름과 이미 열린 창 재사용
```javascript
var openWindows = {};   // 이름 → Window (전역 변수 하나에 덮어쓰지 않음)

function openDocView(docId) {
  var name = 'docView_' + String(docId).replace(/[^A-Za-z0-9]/g, '');  // 편집 창과 다른 이름
  var w = openWindows[name];
  if (w && !w.closed) { w.focus(); return w; }                          // 다시 로드하지 않음
  w = window.open('/doc/view?docId=' + encodeURIComponent(docId), name, 'width=900,height=800');
  if (!w) throw new Error('popup_blocked');                             // null 확인 후 사용
  openWindows[name] = w;
  return w;
}
```

### 2) 비동기 값이 필요할 때: 클릭 즉시 빈 창 먼저
```javascript
button.addEventListener('click', function () {
  var w = window.open('about:blank', 'itemView_' + itemId);   // 클릭 직후 동기 호출
  if (!w) { alert('팝업이 차단되었습니다'); return; }
  fetch('/api/items/' + encodeURIComponent(itemId) + '/board')
    .then(function (r) { return r.json(); })
    .then(function (d) { w.location.href = '/board/view?boardId=' + d.boardId + '&itemId=' + itemId; })
    .catch(function () { w.close(); });
});
```

### 3) 위젯 → 호스트: 메시지로 열기 요청
```javascript
// 호스트(본창)
window.addEventListener('message', function (e) {
  if (e.origin !== 'https://widget.example.com') return;   // 보낸 출처 확인
  var msg = e.data || {};
  if (msg.type === 'open' && /^\d{1,20}$/.test(String(msg.id))) {   // 형식 재검사
    try { openDocView(msg.id); e.source.postMessage({ id: msg.reqId, ok: true }, e.origin); }
    catch (err) { e.source.postMessage({ id: msg.reqId, ok: false, code: 'popup_blocked' }, e.origin); }
  }
});
```

## 주의할 점
- 메시지에 실려 온 주소(url)를 그대로 열지 않는다. ID만 받아 호스트가 주소를 만든다.
- 창 이름에는 영숫자만 쓰는 편이 안전하다. 중괄호·하이픈이 든 ID는 걸러 쓴다.
- 이름 재사용 동작과 opener 처리는 브라우저마다 조금씩 다를 수 있다. 실제 대상 브라우저에서 두 창을 번갈아 열어 확인한다.

## 참고
- MDN: Window.open() (target 이름, noopener), Window.opener, Window.postMessage()
- HTML Living Standard: Browsing context names, Popup blocking (transient activation)
