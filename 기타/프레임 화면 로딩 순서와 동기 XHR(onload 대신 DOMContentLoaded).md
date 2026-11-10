# 프레임 화면 로딩 순서와 동기 XHR(onload 대신 DOMContentLoaded)

> 분류: 기타 · 최근 갱신: 2026-11-09

## 한 줄 요약
왼쪽 트리·오른쪽 목록처럼 프레임을 나눈 화면이 느릴 때는 서버보다 "앞 단계가 끝나야 다음을 시작하는" 순서가 시간을 쌓는 경우가 많다. 목록 조회를 `onload` 대신 문서 준비 시점에 시작하고, 같은 출처 프레임은 메인 스레드를 공유하므로 한쪽의 동기 XHR이 다른 프레임까지 멈춘다는 점을 고려한다.

## 내용
### 시간이 쌓이는 구조
- 흔한 순서: 왼쪽 프레임 `onload` → 트리 그리기·하위 노드 확인(동기 XHR 여러 번) → 기본 항목 클릭 → 오른쪽 프레임 주소 지정 → 오른쪽 문서 로드 → `onload` → 목록 조회 → 그리기.
- `window.onload`는 그 문서의 이미지·iframe 등 모든 하위 리소스가 끝나야 실행된다. 이미지 한 장이 연결 대기로 늦으면 목록 조회 시작도 그만큼 늦는다(측정 사례: 목록 문서 도착 후 조회 시작까지 1.4초).
- 개선: 목록 조회만 `DOMContentLoaded`(jQuery `$(document).ready`)에서 시작하고 응답이 오면 바로 그린다. iframe(미리보기 등)을 건드리는 처리만 `onload` 뒤로 미룬다. 이미 시작했으면 `onload`에서 다시 부르지 않게 플래그를 둔다.

### 동기 XHR이 다른 프레임도 멈춘다
- 같은 출처의 프레임들은 브라우저 메인 스레드 하나를 같이 쓴다. 한 프레임이 `xhr.open(..., false)`로 기다리는 동안 다른 프레임의 JS 실행과 그리기도 멈춘다.
- 그래서 오른쪽 목록을 일찍 띄워도 왼쪽 동기 XHR이 끝나야 그려진다. "프레임 동시 로딩"의 기대 효과는 동기 대기만큼 줄어든다.
- 오래된 트리 컴포넌트는 "요청하고 그 자리에서 결과를 받아 다음 줄에서 쓰는"(노드를 그리다 자식이 있는지 물어 + 아이콘을 정함, 펼친 직후 자식 노드를 찾음) 구조라 비동기로 바꾸면 결과가 오기 전에 다음 줄이 실행돼 깨진다. 공용 파일을 고치지 않고 개선하려면, 그리기 직전에 필요한 결과를 한 번에 받아 두고 같은 이름의 함수를 화면에서 재정의해 받아 둔 결과를 돌려주게 한다(없으면 원래 요청으로).

### 프레임을 미리 띄울 때의 위험
- 왼쪽 트리가 "보이는가"로 권한을 판정하고 오른쪽은 자체 권한만 보면, 두 판정이 다른 사용자에게 목록이 떴다가 "권한 없음"으로 덮이는 깜빡임이 생긴다. 서버가 트리와 같은 메소드로 판정해 확실할 때만 미리 띄우고 최종 판단은 왼쪽에 둔다.
- 왼쪽 자동 클릭이 오른쪽 주소를 다시 넣으면 목록을 두 번 부르고, 그사이 사용자가 오른쪽에서 이동했다면 그 동작이 사라진다. URL 비교 대신 "미리 띄웠음" 표시로 판단한다.
- 효과 추정의 근거가 된 측정이 낡았거나, 왼쪽 시간이 대부분 메인 스레드 스크립트 실행이면 이득이 작다. 성능 기록(Profiler)으로 왼쪽 시간을 동기 XHR·그리기·대기로 나눠 본 뒤 정한다.

## 핵심 코드
### 1) 목록 조회를 문서 준비 시점에 시작
```javascript
var listStarted = false;
function startList() {
  if (listStarted) return;
  listStarted = true;
  loadList();                                  // 응답이 오면 바로 표를 그림
}
document.addEventListener('DOMContentLoaded', startList);
window.addEventListener('load', function () {
  startList();                                 // 앞에서 못 시작한 경우만
  initPreviewFrame();                          // iframe을 만지는 처리는 load 뒤에
});
```

### 2) 공용 트리의 동기 확인을 일괄 결과로 대신하기
```javascript
var prefetched = null;                         // { nodeId: true/false }
function buildTree(rootId) {
  var xml = fetchChildrenSync(rootId);
  prefetched = fetchHasChildrenBatch(collectIds(xml));   // 요청 1번
  try { tree.bind(xml); } finally { prefetched = null; }
}
// 공용 트리가 노드마다 부르는 함수를 이 화면에서 재정의
function hasVisibleChildren(nodeId) {
  if (prefetched && Object.prototype.hasOwnProperty.call(prefetched, nodeId)) {
    return prefetched[nodeId];
  }
  return hasVisibleChildrenSync(nodeId);       // 받아 둔 결과가 없으면 원래 방식
}
```

## 주의할 점
- 메인 스레드 동기 XHR은 브라우저가 사용 중단 예정으로 경고한다. 새 코드에는 쓰지 않는다.
- 측정할 때 주기적으로 도는 폴링(알림 개수 등)이 구간에 겹치면 화면 시간이 부풀려진다. 해당 요청을 빼고 본다.
- Tomcat access log의 `%t`는 기본 설정에서 요청이 들어온 시각이 아니라 **응답이 끝난 시각**이다. 브라우저 쪽 "요청 시작 시각"과 섞어 비교하면 "요청이 늦게 도착했다"고 잘못 읽는다. 서버 처리 시간은 `%D`(또는 `%T`)를 로그 형식에 넣어 따로 본다. "가끔만 느린" 요청은 그 순간 스레드 덤프를 1~2초 간격으로 여러 번 떠서 요청 스레드가 어디서 기다리는지 확인한다.

## 참고
- MDN: DOMContentLoaded, load 이벤트, Synchronous and asynchronous requests
- 관련 노트: [개발자 도구로 iframe 화면 이동 시간 재기](개발자 도구로 iframe 화면 이동 시간 재기.md)
