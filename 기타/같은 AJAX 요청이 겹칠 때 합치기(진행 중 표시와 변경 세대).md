# 같은 AJAX 요청이 겹칠 때 합치기(진행 중 표시와 변경 세대)

> 분류: 기타 · 최근 갱신: 2026-11-11

## 한 줄 요약
여러 프레임·화면 단계가 각자 "최신으로 맞추자"며 같은 무거운 요청(메뉴 건수 등)을 동시에 보내면, 진행 중 표시로 겹치는 호출을 막고 변경 세대 번호로 "보내는 사이 데이터가 바뀌었으면 끝난 뒤 한 번 더"를 보장한다. 단, 요청을 언제 출발시키느냐에 따라 전체 완료 시간이 오히려 늘 수 있으니 측정으로 정한다.

## 내용
### 왜 겹치나
- 왼쪽 메뉴 초기화, 오른쪽 목록 화면 초기화, 목록 응답 처리처럼 서로 다른 시점에 같은 갱신 함수를 부르는 코드가 쌓여 있으면, 한 번의 화면 이동에 같은 요청이 2~3번 나간다.
- 데이터가 적을 때는 수십 ms라 티가 안 나다가, 데이터가 늘어 그 쿼리가 수 초가 되면 겹친 요청이 DB를 다투며 화면이 느려진다.
- 호출 위치를 지우면 그 시점(예: 처리 뒤 목록 재조회)의 갱신이 빠질 수 있다. 위치는 두고 "겹칠 때 한 번만" 나가게 하는 편이 안전하다.

### 동작 규칙
- 모든 호출이 같은 함수(같은 프레임의 전역)를 거치므로 그곳에 상태 변수를 둔다.
- `inFlight`: 요청 중이면 새 요청을 보내지 않는다. 성공·실패·빈 응답 모두에서 반드시 푼다(안 풀면 이후 갱신이 영원히 막힘).
- `dirtyGen`/`sentGen`: 데이터를 바꾼 뒤의 호출은 세대를 올린다. 응답이 왔을 때 `dirtyGen > sentGen`이면 보내는 사이 변경이 있었으니 한 번 더 보낸다. 처리 직후 옛 숫자가 남지 않는다.
- 초기화성 호출(`"init"`)은 진행 중이거나 방금(예: 2초 안) 끝났으면 무시한다. 인자 없는 호출(재조회·처리 뒤)은 항상 최신을 보장한다.
- 요청 객체는 호출마다 새로 만든다. 공용 XHR 객체를 재사용하면 앞 요청의 응답 처리기가 덮인다.

### 출발 시점이 체감 속도를 정한다
- 실측 예: 목록 SQL 3.0초, 건수 SQL 3.7초.
  - 원래(건수 2~3번 동시): 목록 보임 3.1초, 전부 끝 6.2초.
  - 겹침만 막음: 전부 끝 4.0초.
  - "목록 그린 뒤 건수": 목록 보임 3.0초, 전부 끝 **6.7초**(둘이 차례로 돌아 오히려 늘어남).
  - "목록 요청과 동시에 건수 출발": 목록 보임 3.0~3.1초, 전부 끝 3.6~4.0초.
- 무거운 쿼리 하나가 동시에 도는 정도는 목록 시간에 거의 영향이 없었다. 그래서 "뒤로 미루기"보다 "중복 제거 + 동시 출발"이 낫다.
- 그래도 사용자가 보는 목록 시간은 그대로다. 체감을 줄이려면 결국 각 SQL을 빠르게 해야 한다. 요청 합치기의 주된 효과는 서버·DB 부하 감소다.

## 핵심 코드
```javascript
var countInFlight = false;   // 건수 요청 진행 중
var dirtyGen = 0;            // 데이터가 바뀔 때마다 1씩 증가
var sentGen = -1;            // 마지막으로 보낸 요청이 반영하는 세대
var lastDoneAt = 0;

function refreshCount(mode) {                       // 여러 화면이 이 함수만 부른다
    if (mode !== "init") dirtyGen++;                // 처리 뒤 호출 = 변경 표시
    if (countInFlight) return;                      // 끝난 뒤 onDone이 필요하면 다시 보냄
    if (mode === "init" && Date.now() - lastDoneAt < 2000) return;   // 초기화 중복 무시
    sendCount();
}

function sendCount() {
    countInFlight = true;
    sentGen = dirtyGen;
    var xhr = new XMLHttpRequest();                 // 요청마다 새 객체
    xhr.open("POST", "/api/menu-counts", true);
    xhr.onreadystatechange = function () {
        if (xhr.readyState !== 4) return;
        try { if (xhr.status === 200 && xhr.responseText) renderCounts(JSON.parse(xhr.responseText)); }
        catch (e) { /* 빈 응답·파싱 실패도 아래에서 해제 */ }
        countInFlight = false;
        lastDoneAt = Date.now();
        if (dirtyGen > sentGen) sendCount();        // 보내는 사이 변경이 있었으면 한 번 더
    };
    xhr.send();
}

// 목록을 요청하는 순간 건수도 함께 출발 (응답 처리 쪽 호출은 제거)
function loadList(params) {
    requestCountFromMenu();                          // 다른 프레임 함수가 아직 없으면 잠시 재시도
    ajaxList(params, renderList);
}
function requestCountFromMenu(tries) {
    var f = parent.frames["left"];
    if (f && typeof f.refreshCount === "function") { f.refreshCount(); return; }
    if ((tries || 0) < 20) setTimeout(function () { requestCountFromMenu((tries || 0) + 1); }, 250);
}
```

## 주의할 점
- 바꾸기 전에 그 함수를 부르는 모든 경로(팝업 닫기, 삭제 뒤 목록만 다시 부르기, 미리보기, 대시보드 등)를 목록으로 만들고 경로별로 요청 횟수와 최종 값을 시험한다.
- 다른 프레임 함수를 부를 때는 그 프레임이 아직 안 떴거나 그 함수가 없는 화면(관리자 화면 등)이 있다. 존재 확인 후 부른다.
- 측정은 "목록이 보이는 시간"과 "모든 요청이 끝난 시간"을 따로 잰다. 둘 중 하나만 보면 개선 효과를 잘못 보고한다.

## 참고
- 관련 노트: [목록과 건수를 따로 요청할 때 응답 순서 버그](목록과 건수를 따로 요청할 때 응답 순서 버그.md), [달력 화면의 중복 동기 요청 없애기와 받은 응답 재사용](달력 화면의 중복 동기 요청 없애기와 받은 응답 재사용.md)
