# jQuery Mobile 화면에서 무거운 라이브러리 지연 로드

> 분류: 기타 · 최근 갱신: 2026-11-04

## 한 줄 요약
jQuery Mobile 앱에서 한 화면만 쓰는 차트 같은 큰 라이브러리를 공통 `<head>`에서 빼고, 그 화면의 `pageshow`에서 한 번만 받아 실행하면 첫 로드가 가벼워진다. AJAX 페이지 이동은 `<head>`와 page 섹션 바깥을 버리므로 필요한 값은 섹션 안에 두고, 요소는 활성 페이지 범위에서 찾는다.

## 내용
### jQuery Mobile AJAX 이동의 특성
- `$.mobile.changePage(url)`는 새 문서에서 `data-role="page"` 섹션만 떼어 DOM에 넣는다. 새 문서의 `<head>`와 섹션 바깥 스크립트는 실행되지 않는다. 그래서 공통 import에 넣은 JS·CSS 비용은 첫 전체 로드(로그인 화면, 새로고침) 때 한 번 든다.
- 여러 페이지가 DOM에 함께 있고, 같은 화면을 다시 불러오면 같은 id의 페이지가 잠깐 두 개가 된다. `getElementById`나 범위 없는 `$("#id")`는 다른 페이지의 요소를 잡을 수 있다. `$(this)`(pageshow의 페이지)나 `.ui-page-active` 범위로 찾는다.
- 같은 화면 새로고침 때 새 페이지의 `pageshow`가 이전 페이지의 `pagehide`보다 먼저 올 수 있다. 전역(window) 핸들러는 새로 붙이기 전에 이전 것을 먼저 뗀다.

### 쓰지 않는 의존 파일 찾기
- 라이브러리 메이저 버전이 바뀌면 의존 파일이 달라진다. 예: 차트 라이브러리 3.x는 그래픽·유틸 라이브러리를 따로 로드해야 하지만 4.x는 내장한다. 다른 화면에서 복사해 온 import에 옛 의존 파일이 남아 있기 쉽다.
- 파일 정체는 파일 첫 줄 배너로 확인하고, 사용처는 전역 이름(직접 사용, `parent.`/`top.` 경유)으로 저장소 전체를 검색한다. 전역 폴리필(core-js 등)은 다른 코드가 몰래 기대고 있을 수 있어 함부로 빼지 않는다.

### 지연 로더 설계
- 이미 로드됐으면 바로 끝, 받는 중이면 진행 중인 요청을 같이 기다린다(Deferred 하나 공유). 실패하면 다음에 다시 시도할 수 있게 비운다.
- CSS는 id를 붙여 한 번만 넣는다.
- `$.ajax({dataType:"script"})`는 같은 출처면 XHR로 받아 eval로 실행한다. CSP가 `unsafe-eval`을 막으면 실행되지 않고, 오류 스택에 파일 이름이 안 나온다. `crossDomain:true`를 주면 `<script src>` 태그 방식으로 실행하고 끝나면 태그를 뗀다.
- 앱 전역 AJAX 타임아웃이 사실상 무제한이면 이 요청만 따로 타임아웃(예: 30초)을 둔다.
- 버튼 연결처럼 라이브러리와 무관한 초기화는 즉시 하고, 그리기만 로드 후에 한다. 로드 중 다른 화면으로 나갔으면 그리지 않고 로딩 표시를 닫는다.
- 버전 쿼리(`?v=`)가 붙은 주소는 서버에서 만들어 page 섹션 안 숨김 값으로 넘긴다.

## 핵심 코드
### 1) 섹션 안 숨김 값 (JSP)
```jsp
<div data-role="page" id="statsPage">
  <input type="hidden" id="chartJsSrc" value="<%= ctx %>/js/lib/chart.min.js?v=<%= ver %>"/>
  <input type="hidden" id="chartCssSrc" value="<%= ctx %>/css/lib/chart.min.css?v=<%= ver %>"/>
  ...
</div>
```

### 2) 로더
```javascript
var chartLibLoading = null;
function loadChartLibs(page) {
  if (typeof Chart !== "undefined") return $.Deferred().resolve().promise();
  if (chartLibLoading) return chartLibLoading;
  if (!document.getElementById("chartCss")) {
    $("<link>", { id: "chartCss", rel: "stylesheet", href: page.find("#chartCssSrc").val() }).appendTo("head");
  }
  chartLibLoading = $.ajax({
    url: page.find("#chartJsSrc").val(),
    dataType: "script",
    cache: true,
    crossDomain: true,          // script 태그 방식: eval·CSP 문제 회피
    timeout: 30000
  }).fail(function () { chartLibLoading = null; });   // 다음 진입 때 재시도
  return chartLibLoading;
}
```

### 3) 화면 진입
```javascript
$(document).on("pageshow", "#statsPage", function () {
  var page = $(this);                                  // 이 페이지 안에서만 찾는다
  page.find("#downloadBtn").off("click").on("click", exportData);   // 즉시 연결
  $.mobile.loading("show");
  loadChartLibs(page).done(function () {
    if (!page.hasClass("ui-page-active")) return;      // 그사이 나갔으면 그리지 않음
    drawCharts(page);
  }).fail(function () {
    if (page.hasClass("ui-page-active")) page.find(".content").append($("<p>").text("오류가 발생했습니다."));
  }).always(function () {
    if (page.hasClass("ui-page-active")) $.mobile.loading("hide");
  });
});
$(document).on("pagehide", "#statsPage", function () { $.mobile.loading("hide"); });
```

## 주의할 점
- 전체 로드(주소 직접 열기)와 AJAX 이동 두 경로를 모두 확인한다. 전체 로드에서는 전역 변수가 초기화되어 다시 받는 것이 정상이다.
- 확인 항목: 첫 진입 1회만 받는지, 다시 들어올 때 0회인지, script 태그·CSS link·핸들러가 쌓이지 않는지, 로드 실패와 받는 도중 나가기.
- 첫 진입 때 받는 양이 늘어나므로 느린 망에서는 로딩 표시가 필요하다.

## 참고
- jQuery Mobile API: Page events (pagecreate, pageshow, pagehide), `$.mobile.loading`
- jQuery API: `jQuery.ajax` (dataType "script", crossDomain)
- MDN: Content-Security-Policy `script-src` 'unsafe-eval'
