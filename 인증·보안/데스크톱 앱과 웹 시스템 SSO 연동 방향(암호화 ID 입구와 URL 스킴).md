# 데스크톱 앱과 웹 시스템 SSO 연동 방향(암호화 ID 입구와 URL 스킴)

> 분류: 인증·보안 · 최근 갱신: 2026-11-25

## 한 줄 요약
메신저 같은 데스크톱 앱과 웹 시스템의 SSO는 "앱 → 웹"(앱이 웹 화면을 로그인된 채로 연다)과 "웹 → 앱"(웹이 앱을 실행·로그인시킨다)이 서로 다른 기능이다. 앞쪽은 흔히 암호화한 사용자 ID를 받는 SSO 입구로, 뒤쪽은 앱이 등록한 사용자 정의 URL 스킴으로 만든다.

## 내용
- **방향부터 가른다.** "메신저 SSO가 있다"고 해도 대개 앱 → 웹 한쪽만 있다. 기존 코드를 볼 때 다음을 나누어 찾는다.
  - 앱 → 웹: 앱이 호출하는 웹 쪽 입구 URL(예: `/messenger/sso`). 화면 종류 파라미터로 메일·결재·포털 같은 목적 화면을 고른다.
  - 앱 → 웹 로그인 검증: 앱 로그인 창이 ID·비밀번호를 웹 시스템에 물어보는 API. 결과를 `OK`·`PASSWORD_EXPIRED`·`STOPPED`·`FAIL` 같은 코드로 돌려준다.
  - 웹 → 앱 알림: 웹 시스템이 메신저 서버 API를 불러 알림을 보낸다(로그인과는 무관).
  - 웹 → 앱 실행: 웹 화면의 버튼으로 앱을 띄운다. 이게 없으면 "설치 파일 다운로드" 버튼만 있는 경우가 많다.
- **암호화 ID 입구의 동작.** 앱이 사용자 ID를 공유 키(AES 등)로 암호화해 넘기면, 웹은 복호화해서 사용자가 있는지·정지 사용자가 아닌지만 확인하고 로그인 세션(쿠키)을 만든다. 비밀번호 칸을 받더라도 검증에 쓰지 않는 구현이 흔하다. 즉 **복호화가 성공하면 곧 로그인**이므로 보안은 전적으로 키와 메시지 형식에 달려 있다.
- **암호화 ID 입구에서 점검할 것.**
  - 같은 암호문을 다시 보내면 계속 로그인되는가(재전송). 발급 시각을 함께 넣고 짧은 유효시간(예: 1~2분)을 넘으면 거부한다.
  - 1회용 값(nonce)을 넣고 서버에 쓴 값을 기록해 두 번째 사용을 막는다. 서버가 여러 대면 이 기록은 DB 같은 공용 저장소에 둔다.
  - 암호화만 하고 무결성 검사가 없으면(예: AES-CBC 단독) 암호문을 조작하는 공격 여지가 있다. AES-GCM처럼 인증 태그가 있는 방식이나 HMAC 서명을 붙인다.
  - 키를 앱 배포본에 그대로 넣으면 앱을 분석해 키를 꺼낼 수 있다. 가능하면 앱이 서버에서 짧은 1회용 코드를 받아 그 코드를 넘기는 방식이 낫다([1회용 코드 SSO와 refresh token 로테이션](1회용 코드 SSO와 refresh token 로테이션.md)).
  - 암호문을 URL 쿼리에 실으면 접근 로그·브라우저 기록에 남는다. 유효시간이 짧은 1회용이어야 남아도 위험이 적다.
- **웹 → 앱 실행은 사용자 정의 URL 스킴으로.** 앱 설치 프로그램이 OS에 `myapp://` 같은 스킴을 등록하면, 웹 페이지가 그 주소로 이동할 때 브라우저가 "앱 열기" 확인을 띄우고 앱이 주소 전체를 인자로 받는다. 자동 로그인까지 하려면 주소에 위의 1회용 코드를 실어 앱이 서버에 검증받게 한다.
- **옛 방식(ActiveX)은 지금 브라우저에서 안 된다.** IE 시절에는 `new ActiveXObject(...)`로 메신저 상태 표시·대화창 열기·PC의 exe 실행을 했는데, Chrome·Edge에는 ActiveX가 없어 동작하지 않는다. 코드에 남아 있어도 현재 기능으로 치면 안 된다.

## 핵심 코드
웹 → 앱 실행(브라우저). 스킴이 없으면 아무 일도 일어나지 않으므로 잠시 뒤 설치 안내를 띄운다.

```javascript
// myapp:// 는 앱 설치 프로그램이 등록한 스킴으로 바꾼다
function openDesktopApp(oneTimeCode) {
  var url = 'myapp://open?code=' + encodeURIComponent(oneTimeCode);
  var timer = setTimeout(function () {
    if (!document.hidden) alert('앱이 설치되어 있지 않으면 먼저 설치해 주세요.');
  }, 1500);
  window.addEventListener('blur', function () { clearTimeout(timer); }, { once: true });
  window.location.href = url;
}
```

Windows에서 스킴 등록(앱 설치 프로그램이 할 일, 현재 사용자 범위).

```powershell
# <앱 경로>는 실제 실행 파일 경로로 바꾼다
$base = 'HKCU:\Software\Classes\myapp'
New-Item -Path "$base\shell\open\command" -Force | Out-Null
Set-ItemProperty -Path $base -Name '(default)' -Value 'URL:MyApp Protocol'
Set-ItemProperty -Path $base -Name 'URL Protocol' -Value ''
Set-ItemProperty -Path "$base\shell\open\command" -Name '(default)' -Value '"<앱 경로>\MyApp.exe" "%1"'
```

앱 → 웹 SSO 입구에서 유효시간·1회용 검사(서버, 개념 예시).

```java
// payload = 복호화·서명 검증을 마친 {userId, issuedAt, nonce}
SsoPayload p = ssoCrypto.decryptAndVerify(token);     // AES-GCM 등 인증 태그가 있는 방식
long age = System.currentTimeMillis() - p.getIssuedAt();
if (age < 0 || age > 120_000) throw new SsoRejectedException("expired");
// 공용 DB에 nonce를 유니크 키로 넣어 두 번째 사용을 막는다(서버 여러 대 대비)
if (!usedNonceRepository.insertIfAbsent(p.getNonce())) throw new SsoRejectedException("replayed");
User user = userService.findActiveUser(p.getUserId());
if (user == null || user.isStopped()) throw new SsoRejectedException("user");
loginService.createLoginSession(request, response, user);
```

## 주의할 점
- "SSO 있음"이라는 답을 들으면 어느 방향인지 먼저 확인한다. 앱 → 웹 입구가 있다고 웹에서 앱을 띄울 수 있는 것은 아니다.
- 사용자 정의 스킴은 앱 쪽 지원이 있어야 한다. 앱이 스킴을 등록하는지, 받은 인자로 무엇을 하는지는 앱 제작사에 확인한다.
- 비밀번호를 검사하지 않는 입구는 키가 새면 모든 계정으로 로그인할 수 있는 문이 된다. 유효시간·1회용·무결성 검사를 함께 둔다.
- 오래된 ActiveX 코드는 지금 브라우저에서 동작하지 않으므로 기능 목록에서 빼고 판단한다.

## 참고
- Microsoft 문서: Registering an Application to a URI Scheme
- OWASP: Session Management Cheat Sheet(재전송 방지)
