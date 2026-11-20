# 여러 비동기 응답을 모아 한 번 보내기와 화면 이탈·bfcache 처리

> 분류: 기타 · 최근 갱신: 2026-11-19

## 한 줄 요약
여러 영역을 각각 비동기로 그리는 화면에서 그 결과를 다른 곳(상위 창의 위젯 등)에 한 번에 보내려면, 영역별 완료 표시 + 상한 타이머 + 조회 회차 번호 + 화면 이탈 차단 + bfcache 복원 처리를 함께 둬야 한다. 하나라도 빠지면 일부만 담긴 정보, 떠난 뒤 재전송, 이전 조회 응답 섞임이 생긴다.

## 내용
- 응답이 오는 대로 짧은 지연(예: 300ms) 뒤 보내는 방식은 응답 간격이 벌어지면 여러 번 보내고, 그 사이 받는 쪽이 일부만 담긴 정보를 들고 있게 된다. → 필요한 영역이 **모두 끝났을 때** 한 번 보낸다. 실패한 영역도 "끝남"으로 기록하고 `hasMore` 같은 표시를 켠다.
- 응답이 영영 안 오는 영역에 대비해 **조회 시작 시점부터** 잰 상한(예: 5초) 뒤 모인 것만 보낸다. 첫 응답 시점부터 재면 상한이 밀린다.
- 다시 조회(창을 닫고 새로고침 등)할 때 이전 회차 요청의 늦은 응답이 새 수집에 섞인다. → 조회마다 회차 번호를 올리고 콜백에서 번호가 다르면 화면에는 그리되 수집에서는 뺀다.
- 화면을 떠날 때 상위 창에서 정보를 비우는 처리(`beforeunload`)가 있어도, 그 뒤 늦게 온 응답이나 대기 타이머가 다시 보낼 수 있다. → `beforeunload`·`pagehide`에서 대기 타이머를 지우고 이후 전송을 막는 플래그를 세운다.
- 뒤로 가기 캐시(bfcache)로 복원되면 스크립트 상태가 그대로 되살아나 "전송 막힘" 상태로 남는다. → `pageshow`의 `event.persisted`가 true면 막힘을 풀고 다시 보낸다. 덜 모인 채 복원됐으면 모인 것만 바로 보낸다.
- 루프 안에서 행마다 정하는 값(예: "내 차례" 여부)은 행 시작마다 초기화한다. 바깥 변수를 재사용하면 앞 행 값을 물려받는다.
- 받는 쪽(위젯)이 없을 때는 수집 자체를 하지 않도록 처음 쓸 때 한 번 판정한다. 최상위 창 접근은 다른 출처 iframe 안에서 보안 예외가 나므로 try/catch로 감싼다.

## 핵심 코드
```javascript
var gen = 0, done = {}, items = [], blocked = false, sendTimer = null, capTimer = null;
var SECTIONS = ['counts', 'late', 'doing', 'dept', 'draft', 'end'];

function getWidget() {                       // 다른 출처 창 안이면 접근 자체가 예외
  try { return (window.top && window.top.Widget) || null; } catch (e) { return null; }
}
function startCollect() {                    // 화면 로드·재조회 때
  gen++; done = {}; items = []; blocked = false;
  clearTimeout(sendTimer); clearTimeout(capTimer);
  capTimer = setTimeout(function () { send(true); }, 5000);   // 조회 시작부터 상한
  return gen;
}
function sectionDone(myGen, name, rows, failed) {
  if (myGen !== gen || !getWidget()) return; // 이전 회차 응답·위젯 없음은 무시
  if (rows) items = items.concat(rows);
  done[name] = failed ? 'fail' : 'ok';
  if (SECTIONS.every(function (s) { return done[s]; })) {
    clearTimeout(sendTimer);
    sendTimer = setTimeout(function () { send(false); }, 300);
  }
}
function send(partial) {
  if (blocked) return;
  clearTimeout(capTimer);
  var failed = SECTIONS.some(function (s) { return done[s] !== 'ok'; });
  var w = getWidget();
  if (w) w.setContext({ items: items.slice(0, 20), hasMore: partial || failed });
}
function stop() { blocked = true; clearTimeout(sendTimer); clearTimeout(capTimer); }
window.addEventListener('beforeunload', stop);
window.addEventListener('pagehide', stop);
window.addEventListener('pageshow', function (e) {
  if (e.persisted) { blocked = false; send(!SECTIONS.every(function (s) { return done[s]; })); }
});
```

## 주의할 점
- 화면을 떠날 때 정리하는 공통 장치가 이미 있는지 먼저 찾는다. 장치가 없다고 단정하고 새 기록 장치를 만들었다가, 실제 원인은 일부 화면에 그 장치 include가 빠진 것뿐이었던 경우가 있다.
- 실패 응답을 잡는 곳(ajax `error`, 응답 처리 `catch`, 빈 응답)을 모두 "끝남"으로 기록하지 않으면 상한 타이머까지 기다리게 된다.
- `unload` 이벤트는 브라우저가 폐지 중이므로 `pagehide`를 함께 쓴다([Chrome unload 이벤트 폐지와 Permissions policy 경고](Chrome unload 이벤트 폐지와 Permissions policy 경고.md)).
- 관련: [같은 AJAX 요청이 겹칠 때 합치기(진행 중 표시와 변경 세대)](같은 AJAX 요청이 겹칠 때 합치기(진행 중 표시와 변경 세대).md)
