# 숨은 iframe으로 다른 시스템 SSO 입구를 먼저 호출하기

> 분류: 인증·보안 · 최근 갱신: 2026-11-18

## 한 줄 요약
주소에 실은 토큰(`/sso/index.jsp?t=<토큰>`)으로만 로그인하는 다른 시스템의 상세 화면을 새 창으로 열 때, 입구가 상세 주소로 보내 주는 기능(`returnUrl`)이 없으면 브라우저 안 숨은 iframe으로 SSO 입구를 먼저 불러 세션 쿠키를 만든 뒤 새 창을 상세로 보낸다. 끝 판정, 시간 상한, 프레임 탈출 위험을 함께 다뤄야 한다.

## 내용
### 서버가 부르면 안 되는 이유
- SSO 입구를 앱 서버(WAS)가 호출하면 서버가 로그인될 뿐 사용자 브라우저에는 상대 시스템 세션 쿠키가 생기지 않는다. 반드시 브라우저에서 불러야 한다.
- SSO 토큰 쿠키가 HttpOnly면 화면 JS가 읽지 못한다. 서버가 쿠키를 읽어 화면에 넘긴다.

### 언제 부를까
- 로그인 때 한 번: 상대 세션이 먼저 만료되면 다시 로그인 화면, 안 쓰는 사용자도 매번 호출, 로그아웃 뒤 상대 세션이 남음.
- 링크를 누를 때(권장): 빈 창을 즉시 열고(팝업 차단 회피) → 요청자에게 바로 성공 응답(호출 시한 회피) → 숨은 iframe으로 SSO 입구 → 끝나면 빈 창을 상세로.

### 호출 방법
- `fetch(url, {mode:"no-cors", credentials:"include"})`: 첫 응답이나 302에서 세션 쿠키를 주면 충분. 다만 응답을 읽을 수 없고 서드파티 쿠키 정책 영향을 받는다.
- 숨은 iframe: JS 이동·폼 자동 제출로 여러 단계를 거쳐야 세션이 생기는 경우까지 처리.

### 끝 판정
- HTTP 302 연쇄는 네트워크에서 처리돼 마지막 화면이 다 뜬 뒤 `onload` 한 번, JS 이동은 단계마다 `onload`.
- `onload`마다 짧은 타이머(예: 0.5초)를 다시 걸어 "마지막 화면 뒤 조용함"을 끝으로 보고, 응답이 없어도 상한(예: 5초)에서 진행한다. 끝 처리는 한 번만.
- iframe은 바로 지우지 말고 조금 뒤(예: 10초) 지운다. 남은 단계가 끊기지 않게.
- `X-Frame-Options: DENY`여도 응답의 Set-Cookie는 저장된다(표시만 막힘).

### 위험과 대안
- 단계 사이 지연이 판정 시간보다 길면(“로그인 중…” 1초 표시 뒤 이동) 중간 화면을 끝으로 오판한다.
- 메인 화면이 iframe 안에서 실제로 실행된다. 프레임 탈출 코드(`if (top != self) top.location = self.location`)가 있으면 앱 화면이 상대 시스템으로 바뀐다 → `sandbox="allow-scripts allow-forms allow-same-origin"`(top 이동·팝업 불허).
- 상대 시스템이 `returnUrl`을 지원하면 `입구?t=<토큰>&returnUrl=<encodeURIComponent(상세)>`로 한 번에 여는 쪽이 단순하다. 이때 넘길 주소가 입구 자체면 `returnUrl`을 붙이지 않는다(로그인 뒤 다시 입구로 오는 무한 루프). 상대 쪽이 값을 디코드하는지 먼저 맞춘다.
- 토큰을 주소에 붙일 때 인코딩 여부는 상대가 `getParameter`로 읽는지(인코딩해야 `+`가 공백이 안 됨) 원문 문자열을 자르는지에 따라 다르다. 같은 시스템에 이미 쓰는 링크가 있으면 그 방식에 맞춘다.

## 핵심 코드
### 1) 서버: HttpOnly 토큰 쿠키를 화면에 넘기기 (Spring)
```java
Cookie c = WebUtils.getCookie(request, "sso_token");    // 쿠키 이름은 환경에 맞게
request.setAttribute("ssoToken", c != null ? c.getValue() : null);
```

### 2) 브라우저: 링크 클릭 → 빈 창 → SSO 선호출 → 상세 이동
```javascript
function openWithSso(detailUrl, ssoEntryUrl) {
  var win = window.open('/blank.html', '_blank');        // 클릭 직후 동기 호출
  if (!win || win.closed) throw { code: 'INTERNAL', message: 'popup_blocked' };
  win.opener = null;
  if (!ssoEntryUrl) { win.location.replace(detailUrl); return; }
  callSso(ssoEntryUrl, function () { win.location.replace(detailUrl); });
}

function callSso(url, done) {
  var frame = document.createElement('iframe'), finished = false, quiet = null, max = null;
  function finish() {
    if (finished) return;
    finished = true; clearTimeout(quiet); clearTimeout(max);
    done();
    setTimeout(function () { if (frame.parentNode) frame.parentNode.removeChild(frame); }, 10000);
  }
  frame.style.display = 'none';
  frame.setAttribute('sandbox', 'allow-scripts allow-forms allow-same-origin');  // 프레임 탈출 막기
  frame.onload = function () { clearTimeout(quiet); quiet = setTimeout(finish, 500); };
  max = setTimeout(finish, 5000);
  frame.src = url;
  document.body.appendChild(frame);
}
```

### 3) returnUrl 지원 시 주소 만들기 (입구 자체면 붙이지 않음)
```javascript
function ssoUrl(target, entry, token) {
  var u = new URL(target);
  var path = u.pathname.toLowerCase().replace(/\/+$/, '');
  var base = entry + '?t=' + token;
  if (path === '/sso' || path === '/sso/index.jsp') return base;   // 무한 루프 방지
  return base + '&returnUrl=' + encodeURIComponent(target);
}
```

## 주의할 점
- 빈 창에 `about:blank` 대신 같은 출처의 빈 페이지를 쓸 때는, 빈 페이지 로드가 끝나기 전에 이동하면 늦게 끝난 로드가 이동을 덮어쓸 수 있다. 빈 페이지 로드와 SSO가 모두 끝난 뒤 한 번만 이동한다.
- 시험은 앱·SSO·상대 시스템 역할을 같은 사이트의 다른 호스트로 두고 302, JS 여러 단계, X-Frame-Options, 느린 응답 경우를 나눠 상세 요청에 세션 쿠키가 실리는지 본다.
- 서드파티 쿠키 차단 정책(다른 사이트 iframe)에서는 쿠키가 저장되지 않을 수 있다. 앱과 상대 시스템이 같은 상위 도메인(같은 사이트)인지 확인한다.

## 참고
- MDN: iframe sandbox, X-Frame-Options, Window.open(), Fetch credentials
- OWASP: Unvalidated Redirects and Forwards
