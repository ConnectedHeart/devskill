# 개발자 도구로 iframe 화면 이동 시간 재기

> 분류: 기타 · 최근 갱신: 2026-11-04

## 한 줄 요약
화면 로딩 시간은 "어디까지를 완료로 보나"에 따라 방법이 다르다. Chrome Network 탭의 Finish는 최상위 문서 요청이 기록에 있을 때만 나오므로, iframe 안 메뉴 이동은 마지막 요청의 끝 시각이나 Performance 탭 스크린샷으로 잰다. 반복 측정은 Playwright, 서버 처리 시간은 access log `%D`로 본다.

## 내용
### Network 탭
- Disable cache를 켜고 Clear 후 동작하면 하단에 Finish(첫 요청 시작~마지막 응답), DOMContentLoaded, Load가 나온다.
- **Finish가 안 나오는 경우:** Chrome은 최상위 페이지 문서 요청이 기록에 있어야 Finish를 계산한다. 기록을 비운 뒤 iframe 안에서만 요청이 나가면 Finish가 없다. DOMContentLoaded·Load도 최상위 기준이라 iframe 전환에서는 바뀌지 않는다.
- 대신: 컬럼 제목 우클릭 → Waterfall → End Time으로 정렬하고, 마지막으로 끝난 요청의 Timing 탭에서 Started at + Total을 읽는다. 첫 요청의 Started at이 0 근처인지 확인한다.
- 느린 요청 찾기: Time 컬럼 정렬, Timing 탭의 Waiting for server response(TTFB, 서버 처리)와 Content Download(응답 크기).
- 폴링 요청(알림 등)이 섞이면 끝 시각이 늘어나므로 이름으로 걸러 낸다. 3~5회 재서 평균을 낸다.

### Performance 탭
- 설정에서 Screenshots를 켜고 Record(Ctrl+E) → 메뉴 클릭 → 다 뜬 뒤 Stop. 스크린샷 줄에서 클릭 시점부터 목록이 보인 시점까지 드래그하면 구간 시간이 나온다. 요청이 끝난 뒤 JS가 그리는 시간까지 포함된다.

### 콘솔
- `performance.getEntriesByType('navigation')`: 그 프레임 문서의 TTFB·DOMContentLoaded·load(ms, 요청 시작 기준). 콘솔 상단 드롭다운에서 프레임을 고른다.
- `performance.getEntriesByType('resource')`: 요청별 시간.
- 클릭부터 특정 요소가 보일 때까지는 스니펫으로 잰다(아래). 메뉴와 결과가 다른 프레임이면 최상위에서 각 iframe 문서에 클릭 리스너를 걸고 iframe이 다시 그려질 때마다 다시 건다.

### 자동 반복 측정과 서버 시간
- Playwright: 클릭 → `waitForSelector`로 목표 요소가 보일 때까지, `page.on('response')`로 요청별 시간도 기록. 이 API는 Node 스크립트용이라 브라우저 콘솔에서는 동작하지 않는다.
- Tomcat access log 패턴에 `%D`(ms, Tomcat 10+는 `%{ms}T`도 가능)를 붙이면 요청마다 서버 처리 시간이 남는다. 기본 패턴 `%h %l %u %t "%r" %s %b`에는 없다.

## 핵심 코드
### 1) 콘솔: 클릭 → 목표 요소 표시까지 (같은 프레임)
```javascript
(() => {
  const sel = '#list tbody tr';            // 완료로 볼 요소
  let t0 = null;
  document.addEventListener('click', () => { t0 = performance.now(); }, { once: true, capture: true });
  const timer = setInterval(() => {
    const el = document.querySelector(sel);
    if (t0 !== null && el && el.offsetParent !== null) {
      console.log('loading(ms):', Math.round(performance.now() - t0));
      clearInterval(timer);
    }
  }, 20);
})();
```

### 2) 콘솔: 다른 프레임의 메뉴 클릭 → 본문 iframe 로드·표시 (top에서 실행)
```javascript
(() => {
  const DONE_SEL = '#list tbody tr';       // 비우면 iframe load까지만
  let t0 = null;
  const hook = d => { try { d.addEventListener('click', () => { t0 = performance.now(); }, true); } catch (e) {} };
  hook(document);
  document.querySelectorAll('iframe,frame').forEach(f => {
    if (f.id === 'contentFrame') return;
    hook(f.contentDocument);
    f.addEventListener('load', () => hook(f.contentDocument));   // 메뉴 프레임이 다시 그려져도 유지
  });
  const cf = document.getElementById('contentFrame');
  cf.addEventListener('load', () => {
    if (t0 === null) return;
    const start = t0; t0 = null;
    if (!DONE_SEL) return console.log('frame load(ms):', Math.round(performance.now() - start));
    const timer = setInterval(() => {
      const el = cf.contentDocument && cf.contentDocument.querySelector(DONE_SEL);
      if (el && el.offsetParent !== null) {
        clearInterval(timer);
        console.log('visible(ms):', Math.round(performance.now() - start));
      }
    }, 20);
  });
})();
```

### 3) Playwright 반복 측정
```javascript
const t0 = Date.now();
await menuFrame.click('#menuBoard');
await page.frameLocator('#contentFrame').locator('#list tbody tr').first().waitFor({ state: 'visible' });
console.log('loading(ms):', Date.now() - t0);
```

### 4) access log에 처리 시간 남기기 (server.xml)
```xml
<Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
       prefix="localhost_access_log" suffix=".txt"
       pattern="%h %l %u %t &quot;%r&quot; %s %b %D" />
```

## 주의할 점
- 다른 출처(origin)의 iframe 문서에는 접근할 수 없어 스니펫이 동작하지 않는다.
- 목록이 이미 떠 있는 상태에서 다시 조회하면 "기존 행이 사라졌다가 새로 생기는 것"을 기다리도록 조건을 바꾼다.
- 캐시 유무, 첫 회(서버 캐시 빈 상태)와 반복 회차를 나눠 기록한다.

## 참고
- Chrome DevTools 문서: Network features reference (Timing, Waterfall), Performance panel
- MDN: Performance API (`PerformanceNavigationTiming`, `PerformanceResourceTiming`)
- Apache Tomcat: Access Log Valve
- 관련 노트: [Playwright 화면 자동화에서 자주 막히는 지점](Playwright 화면 자동화에서 자주 막히는 지점.md)
