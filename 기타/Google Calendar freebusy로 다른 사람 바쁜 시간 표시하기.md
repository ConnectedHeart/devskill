# Google Calendar freebusy로 다른 사람 바쁜 시간 표시하기

> 분류: 기타 · 최근 갱신: 2026-11-16

## 한 줄 요약
회의 참석자의 구글 일정을 보여 줄 때는 일정 목록(`events.list`)보다 바쁜 시간만 주는 `freebusy.query`가 목적에 맞고 가볍다. 참석자마다 토큰이 다르면 병렬로 짧은 타임아웃을 두고 부르고, 자체 일정은 먼저 그린 뒤 구글 결과를 덧칠한다.

## 내용
### 다른 사람 구글 일정을 보는 방법 비교
| 방법 | 받는 것 | 조건 | 판단 |
|---|---|---|---|
| `freebusy.query` | 바쁜 구간(시작·끝)만 | 조회 대상 캘린더 ID(메일 주소), 대상이 free/busy를 공개 | 회의 잡기용으로 가장 적합 |
| 공유 캘린더 `events.list` | 공유 범위만큼(제목 포함 가능) | 대상이 보는 사람에게 직접 공유 | 사람마다 공유 필요 |
| 서비스 계정 + 도메인 전체 위임 | 조직 전원 전체 | Workspace 관리자 설정 | 권한이 너무 넓음 |
| 대상 본인 토큰으로 대리 조회 | 전체 | 대상이 연동해 토큰 저장 | 남이 보는 데 대한 동의 범위 문제 |

- Workspace(회사 도메인)는 조직 안 free/busy 공개가 보통 기본값이라 바로 되고, 개인 Gmail은 남의 free/busy가 기본으로 안 보여 결과가 비어 온다.
- 캘린더 전체 범위 권한(`.../auth/calendar`)에 free/busy 조회가 포함된다. 읽기 전용만 필요하면 `calendar.freebusy` 범위가 있다.
- 사용자 ID와 토큰만 저장하고 구글 계정 주소를 저장하지 않았다면, 대상 캘린더를 `primary`로 그 사람 토큰으로 부르거나 연동 때 주소를 같이 저장해야 한다.

### events.list보다 가벼운 이유
- 응답이 시작·끝 시각뿐이라 작다. 반복 일정을 구글이 펼쳐 주므로 직접 전개 계산이 필요 없다. 비공개 일정도 처음부터 "바쁨"으로만 온다.
- 대신 제목은 볼 수 없다.

### 호출과 화면 설계
- 참석자마다 토큰이 달라 한 요청으로 묶을 수 없으면, **요청 안에서 만든 스레드 풀(최대 5개 정도)** 로 병렬 호출하고 끝나면 닫는다. 10명 순차 3~5초가 병렬이면 가장 느린 1건만큼이다(추정).
- 연결·읽기 타임아웃을 3~5초로 짧게 둔다(라이브러리 기본 20초). 넘기면 그 사람만 "조회 실패"로 표시한다. 토큰 없는 사람은 호출하지 않고 "미연동"으로 둔다.
- 자체 일정 응답을 먼저 그리고, 구글은 **별도 요청**으로 받아 빈 칸만 덧칠한다. 구글이 느리거나 실패해도 기본 화면은 그대로다.
- 외부 응답을 기다리는 서비스 메소드는 트랜잭션 밖에서 돌게 해 DB 커넥션을 붙잡지 않는다.
- 구글은 UTC 시각을 주므로 보는 사람 시간대로 바꾼다.

### 시간 칸 칠하기에서 잘 틀리는 경계
- 끝 시각은 **포함하지 않는다**. 전날 22:00~오늘 00:00 구간이 오늘 00:00~00:30 칸에 칠해지면 안 된다.
- 초 단위 끝 시각(14:00:30)을 분으로 버리면 걸친 칸이 빠진다. 초까지 계산해 올림한다.
- 대상 인원 상한이 있으면 화면에서 나눠 요청해 뒤쪽 사람이 빠지지 않게 한다.
- 날짜 키를 만드는 기존 함수가 이미 두 자리인 값에 0을 또 붙이는 등(`"09"` → `"009"`) 한 자리 월·일에서만 깨질 수 있다. 1~9월·1~9일 날짜로도 시험한다.
- 날짜를 빠르게 바꾸면 늦게 온 이전 날짜 응답이 새 화면을 덮는다. 요청 때의 날짜와 현재 날짜가 다르면 버린다.

## 핵심 코드
```java
// 한 사람의 바쁜 구간 (Google Calendar API v3 Java 클라이언트)
FreeBusyRequest req = new FreeBusyRequest()
    .setTimeMin(new DateTime(dayStartUtcMillis))
    .setTimeMax(new DateTime(dayEndUtcMillis))
    .setItems(Collections.singletonList(new FreeBusyRequestItem().setId("primary")));
FreeBusyResponse res = calendarFor(userToken).freebusy().query(req).execute();
List<TimePeriod> busy = res.getCalendars().get("primary").getBusy();   // start / end
```
```java
// 참석자별 병렬 호출, 요청 안에서 만든 풀은 반드시 닫는다
ExecutorService pool = Executors.newFixedThreadPool(Math.min(5, users.size()));
try {
    Map<String, Future<List<TimePeriod>>> futures = new LinkedHashMap<>();
    for (UserToken u : users) futures.put(u.getUserId(), pool.submit(() -> queryBusy(u)));
    for (Map.Entry<String, Future<List<TimePeriod>>> e : futures.entrySet()) {
        try { result.put(e.getKey(), Status.ok(e.getValue().get(5, TimeUnit.SECONDS))); }
        catch (TimeoutException | ExecutionException ex) { result.put(e.getKey(), Status.fail()); }
    }
} finally { pool.shutdownNow(); }
```
```java
// 클라이언트 타임아웃 짧게
HttpRequestInitializer init = request -> {
    credential.initialize(request);
    request.setConnectTimeout(5000);
    request.setReadTimeout(5000);
};
```
```javascript
// 30분 칸 번호: 시작은 내림, 끝은 '포함 안 함' + 초까지 올림
function cellRange(startSec, endSec) {          // 그날 0시 기준 초
  const first = Math.floor(Math.max(0, startSec) / 1800);
  const last  = Math.ceil(Math.min(86400, endSec) / 1800) - 1;
  return last >= first ? [first, last] : null;  // 전날~00:00 끝이면 null
}
```

## 주의할 점
- free/busy 공개 여부는 계정 종류(Workspace·개인)와 대상 설정에 달려 있다. 도입 전에 사용자가 어떤 계정으로 연동하는지 확인한다.
- 대상 본인 토큰으로 남의 일정을 대신 조회하는 방식은 기술적으로 되지만 동의 범위를 벗어난다.
- 비공개 일정은 화면에서만 숨기지 말고 서버 응답에서 빼야 한다. 주소를 직접 부르면 제목이 보인다.

## 참고
- Google Calendar API — Freebusy: query
- [Future로 외부 호출 전체 시간 제한하기](../Java·Spring/Future로 외부 호출 전체 시간 제한하기.md)
