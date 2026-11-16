# 웹 문서 편집기의 대용량 리소스 로딩 지연과 캐시(wasm·서비스 워커)

> 분류: 기타 · 최근 갱신: 2026-11-11

## 한 줄 요약
브라우저에서 도는 문서 편집기(WebAssembly 기반 워드프로세서 등)는 창을 열 때마다 wasm·데이터·글꼴 수십 MB를 받는다. 편집기 서버가 `Cache-Control: no-cache`로 주고 304도 안 하면 브라우저 캐시가 전혀 듣지 않아 매번 다시 받는다. 단계별로 재서 "우리 서버 몫"과 "편집기 몫"을 나눈 뒤, 필요 없는 모듈 늦게 불러오기·편집기 재사용·캐시 설정·서비스 워커 중 고른다.

## 내용
### 원인 나누기
- 문서 창 하나를 열 때 단계: 창 HTML → 편집기 틀(iframe) → 편집기 본체(스크립트·wasm·글꼴) → 문서 열기 → 후처리(서명·결재선 등). 개발자 도구 네트워크 탭이나 자동화 스크립트로 단계별 시작·끝 시각과 전송량을 기록한다.
- 측정 예: 우리 서버 요청 합계 약 0.2초, 편집기 준비 약 1초, 문서 열기 0.4~1초, 창마다 19~37MB 전송. 문서 형식(hwp·hwpx 등)별 차이는 없었다 → 문서가 아니라 편집기 로딩이 원인.
- 응답 헤더를 본다. `no-cache`이고 두 번째 요청에도 200(전체 본문)이면 재검증조차 안 되는 상태다.

### 개선 방법(효과 큰 순)
1. **안 쓰는 모듈은 나중에**: 보기·결재 화면에서 수식 편집기처럼 입력 때만 쓰는 모듈(예: 24MB)을 항상 `<script async>`로 불러오면 전송량의 대부분이 된다. 문서를 연 뒤에 동적으로 불러오면 본문 표시 전 전송량이 크게 준다(37→13MB). 빠른 내부망에선 병렬 다운로드라 0.1~0.2초만 줄지만 느린 망·동시 사용자에서 효과가 크다. 그 모듈 없이 해당 요소(수식 등)가 든 문서가 제대로 보이는지 먼저 확인한다.
2. **편집기 재사용**: 목록 미리보기에서 행마다 iframe을 새로 만들지 말고, 편집기는 한 번 띄우고 문서 열기 API로 내용만 바꾼다(재열기 100ms대). 버튼·첨부 등이 창 재생성을 전제로 짜여 있으면 수정 범위가 크다.
3. **편집기 서버 캐시 설정**: 정적 자원에 `Cache-Control: max-age`(또는 ETag/Last-Modified로 304)를 준다. 파일명에 버전·해시가 없는 자원은 업그레이드 후 옛 파일이 남지 않게 기간을 며칠 정도로 짧게.
4. **서비스 워커**: 편집기를 싣는 프레임 페이지가 우리 출처라면, 그 범위에 서비스 워커를 등록해 편집기 서버로 가는 GET 정적 자원을 Cache Storage에 보관할 수 있다. 첫 1회만 받고 이후 0에 가깝다.
5. 보기 전용은 변환본(PDF·이미지)으로 보여 주기 — 변환 서버와 인쇄·텍스트 읽기 기능 재구현이 필요해 설계 수준 변경.

### 서비스 워커 조건
- HTTPS(또는 localhost)에서만 등록된다. http 사이트에서는 등록이 안 될 뿐 깨지지 않는다.
- 편집기 버전을 캐시 이름에 넣어 버전이 바뀌면 새로 받는다.
- 정적 확장자(js·wasm·data·글꼴·이미지)의 GET만 보관한다. 문서 본문·첨부·열기 API는 절대 보관하지 않는다.
- 끄는 스위치(설정값으로 등록 해제)를 둔다. 잘못 배포된 서비스 워커는 사용자 브라우저에 남는다.
- 다른 출처 응답이 CORS 허용이 아니면 불투명(opaque) 응답이 되어 내용 확인이 안 되고 저장 용량을 크게 차지한다. 시제품으로 먼저 확인한다.

## 핵심 코드
### 1) 입력용 모듈을 문서 연 뒤에 불러오기
```html
<script>
  var LAZY = new URLSearchParams(location.search).get("lazyModule") === "1";
  function loadExtraModule() {
    if (window.__extraLoaded) return; window.__extraLoaded = true;
    var s = document.createElement("script");
    s.src = EDITOR_BASE + "/wasm/extra-module.js"; s.async = true;
    document.head.appendChild(s);
  }
  if (!LAZY) loadExtraModule();          // 편집 화면은 지금처럼 바로
</script>
```
```javascript
editor.open(docUrl, function onOpened() {
  afterOpen();
  if (typeof loadExtraModule === "function") loadExtraModule();   // 보기 화면은 본문이 뜬 뒤
});
```

### 2) 편집기 서버(Tomcat) 정적 자원 캐시
```xml
<filter>
  <filter-name>ExpiresFilter</filter-name>
  <filter-class>org.apache.catalina.filters.ExpiresFilter</filter-class>
  <init-param><param-name>ExpiresByType application/wasm</param-name><param-value>access plus 3 days</param-value></init-param>
  <init-param><param-name>ExpiresByType application/javascript</param-name><param-value>access plus 3 days</param-value></init-param>
  <init-param><param-name>ExpiresByType font/woff2</param-name><param-value>access plus 3 days</param-value></init-param>
</filter>
<filter-mapping><filter-name>ExpiresFilter</filter-name><url-pattern>/*</url-pattern></filter-mapping>
```

### 3) 서비스 워커로 정적 자원만 캐시
```javascript
// sw.js — 편집기 프레임 페이지와 같은 경로 범위에 등록
const CACHE = "editor-static-13.0.1";                       // 편집기 버전
const STATIC = /\.(js|wasm|data|woff2?|ttf|png|svg)(\?|$)/;
self.addEventListener("activate", e => e.waitUntil(
  caches.keys().then(ks => Promise.all(ks.filter(k => k !== CACHE).map(k => caches.delete(k))))));
self.addEventListener("fetch", e => {
  const r = e.request;
  if (r.method !== "GET" || !STATIC.test(r.url) || r.url.includes("/open")) return;   // 문서는 통과
  e.respondWith(caches.open(CACHE).then(c => c.match(r).then(hit => hit ||
    fetch(r).then(res => { if (res.ok || res.type === "opaque") c.put(r, res.clone()); return res; }))));
});
```
```javascript
if (window.SW_ENABLED && "serviceWorker" in navigator) navigator.serviceWorker.register("sw.js");
else if ("serviceWorker" in navigator) navigator.serviceWorker.getRegistrations().then(rs => rs.forEach(r => r.unregister()));
```

## 주의할 점
- 운영에서는 편집기 서버가 그룹웨어에서 문서를 직접 받는 등 로컬과 경로가 다를 수 있다. 로컬 측정만으로 운영 지연을 단정하지 않는다.
- 측정은 1쪽짜리 문서만이 아니라 여러 쪽 문서도 포함한다.

## 참고
- MDN: Service Worker API, Cache Storage, Cache-Control
- Apache Tomcat: ExpiresFilter
- 관련 노트: [Spring 정적 리소스 캐시 헤더(cache-period)와 버전 쿼리](../Java·Spring/Spring 정적 리소스 캐시 헤더(cache-period)와 버전 쿼리.md)
