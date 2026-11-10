# 정적 웹앱을 GitHub Pages PWA로 배포하고 홈 화면에 추가하기

> 분류: 기타 · 최근 갱신: 2026-11-09

## 한 줄 요약
서버 처리가 없는 웹앱은 GitHub Pages 공개 저장소에 올리고 manifest와 서비스 워커를 붙이면, 아이폰·안드로이드 모두 앱스토어 없이 "홈 화면에 추가"로 설치해 오프라인으로 쓸 수 있다.

## 내용
- **언제 맞나**: 처리(파일 해석, 재생, 계산)가 모두 브라우저 안에서 끝나고 서버는 정적 파일만 내려주면 되는 앱. 서버·DB 비용이 없고 사용자가 늘어도 정적 파일 전송량만 는다.
- **GitHub Pages**: 공개 저장소는 무료다. 사이트 1GB, 월 전송량 100GB 정도의 권장 한도가 있다. 개인·비영리 용도 기준이며, 돈을 받는 서비스 운영은 약관에 맞지 않는다. 저장소 Settings → Pages에서 Source "Deploy from a branch", Branch `main` / `(root)`로 켜면 1~2분 뒤 `https://<계정>.github.io/<저장소>/`가 열린다. 새 공개 저장소 생성은 계정 주인이 하는 것이 안전하다.
- **PWA 구성**: `manifest.webmanifest`(이름, 아이콘, `display: standalone`, `start_url`) + 서비스 워커(앱 파일 캐시). 서비스 워커는 https(또는 localhost)에서만 등록된다. 앱 파일 목록을 캐시해 두면 첫 실행 뒤 인터넷 없이 열리고, 열 때마다 바뀐 파일만 확인한다. 새 버전은 "한 번 열어 받은 뒤 다음 실행부터" 적용되는 것이 일반적이다(캐시 이름에 버전을 넣어 교체).
- **사용자 데이터 저장**: 사용자가 넣은 파일(음원 등)은 IndexedDB에 저장한다. 서버로 가지 않는다.
- **아이폰**: Safari로 열고 공유(□↑) → "홈 화면에 추가". iOS는 홈 화면 앱과 Safari 탭의 저장소가 따로라, 홈 화면 아이콘으로 저장한 데이터는 Safari 주소로 열면 보이지 않는다. 항상 아이콘으로 열도록 안내한다. 무음 스위치가 켜져 있으면 Web Audio 소리가 안 날 수 있다.
- **안드로이드**: 크롬 ⋮ → "홈 화면에 추가"(또는 "앱 설치"). 같은 링크 하나로 두 OS에 배포할 수 있다.
- **설치형 APK가 필요할 때**: 같은 웹 코드를 웹뷰 래퍼(Capacitor·Cordova 등)로 감싸 APK를 만든다. Android Gradle 빌드는 프로젝트 경로에 한글 같은 비 ASCII 문자가 있으면 실패할 수 있어, ASCII 경로로 복사해 빌드하는 스크립트를 두면 편하다. 최신 Android Gradle 플러그인은 JDK 17을 요구하므로 기존 JDK를 바꾸지 않고 별도 경로에 설치해 그 빌드에만 `JAVA_HOME`을 준다. 마이크 등을 쓰면 `AndroidManifest.xml`에 권한을 추가한다.
- **APK를 다시 배포하지 않고 웹뷰 앱 갱신하기**: 앱이 열릴 때와 다른 앱에서 돌아올 때 웹 서버의 버전 파일을 확인하고, 새 버전이면 앱 파일들을 받아 기기 저장소에 넣은 뒤 그 버전으로 화면을 다시 연다. 재생·녹음 중에는 미루고, 받아 둔 버전은 오프라인에서도 쓰며, 사용자 데이터(IndexedDB 등)는 그대로 둔다. 이후 APK 재배포는 앱 권한·이름 같은 네이티브 쪽을 바꿀 때만 필요하다. PWA도 서비스 워커 갱신을 감지해 바로 새로 고치게(`skipWaiting`·`clients.claim` + `controllerchange` 때 reload) 하면 "다음 실행부터 적용"을 줄일 수 있다.
- **폰 화면에서 겪은 것**
  - 설정 드롭다운을 바꿀 때마다 무거운 화면(악보 등)을 다시 그리면, 폰에서 수 초 멈추는 동안 다음 터치가 사라진다. 설정창이 열린 동안은 값만 모으고 닫을 때 한 번만 다시 그린다.
  - 인앱 브라우저(메신저 안 브라우저)의 위아래 도구막대 때문에 콘텐츠 영역이 작아져 내용 아래가 잘릴 수 있다. 영역 크기가 바뀔 때마다(`ResizeObserver`) 페이지를 다시 나누고, 한 단이 영역보다 크면 배율을 자동으로 줄인다.
  - 가로로 미는 버튼 줄은 `overflow-x: auto`로 만들고, 마우스로 끌기도 직접 처리하며, 끌다 손을 뗀 자리의 버튼은 클릭으로 처리하지 않는다(이동 거리 기준).
  - 숨은 캔버스에 한 번 더 그려 픽셀을 읽는 배치 계산은 비싸다(그리기 시간의 약 70% 사례). 좌표 계산으로 대신하고 실패할 때만 원래 방식으로 돌아가면 수 배 빨라진다.
- **비용 비교**: 웹앱·APK 직접 배포는 무료, 구글 플레이 등록 $25(1회), 애플 개발자 계정 연 $99, 개인 도메인 연 1~2만 원 정도.

## 핵심 코드
```json
// manifest.webmanifest - name·아이콘 경로를 바꿔 쓴다
{
  "name": "My Practice App",
  "short_name": "Practice",
  "start_url": "./",
  "scope": "./",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#c62828",
  "icons": [
    { "src": "icons/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "icons/icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

```html
<!-- index.html head - iOS는 apple-touch-icon을 홈 화면 아이콘으로 쓴다 -->
<link rel="manifest" href="manifest.webmanifest">
<link rel="apple-touch-icon" href="icons/icon-192.png">
<meta name="apple-mobile-web-app-capable" content="yes">
<script>
  if ('serviceWorker' in navigator) navigator.serviceWorker.register('sw.js');
</script>
```

```javascript
// sw.js - 버전을 올리면 새 캐시로 교체된다
const CACHE = 'app-v3';
const FILES = ['./', 'index.html', 'app.js', 'style.css', 'lib/score-lib.min.js'];
self.addEventListener('install', e => e.waitUntil(caches.open(CACHE).then(c => c.addAll(FILES))));
self.addEventListener('activate', e => e.waitUntil(
  caches.keys().then(keys => Promise.all(keys.filter(k => k !== CACHE).map(k => caches.delete(k))))));
self.addEventListener('fetch', e => e.respondWith(
  caches.match(e.request).then(hit => hit || fetch(e.request))));
```

```javascript
// 웹뷰 앱 자동 갱신 골격 - version.json·파일 목록·저장 함수는 앱에 맞게
async function checkUpdate() {
  if (isPlayingOrRecording()) return;                       // 사용 중이면 다음 기회에
  const remote = await fetch('version.json?t=' + Date.now(), { cache: 'no-store' }).then(r => r.json());
  if (remote.version === localStorage.getItem('appVersion')) return;
  for (const f of remote.files) {                           // 앱 파일만 받아 기기 저장소에
    await saveFile(f, await fetch(f + '?v=' + remote.version).then(r => r.text()));
  }
  localStorage.setItem('appVersion', remote.version);
  reloadFromLocalFiles();                                   // 새 버전으로 화면 다시 열기
}
window.addEventListener('load', checkUpdate);
document.addEventListener('visibilitychange', () => { if (!document.hidden) checkUpdate(); });
```

```javascript
// 설정창: 값만 모았다가 닫을 때 한 번 다시 그리기
let pending = {};
settingsSelects.forEach(sel => sel.addEventListener('change', () => {
  pending[sel.name] = sel.value;
  sel.blur();                                               // 포커스 해제
}));
closeButton.addEventListener('click', () => {
  closeSettings();
  if (Object.keys(pending).length) { applySettings(pending); redrawOnce(); pending = {}; }
});
```

```powershell
# 한글 경로 프로젝트를 ASCII 경로로 복사해 APK 빌드 (경로·JDK 위치를 바꿔 쓴다)
$src = "<한글 경로>\my-app"; $dst = "$env:USERPROFILE\.my-app-build"
robocopy $src $dst /MIR /XD node_modules .gradle build | Out-Null
$env:JAVA_HOME = "$env:USERPROFILE\.jdks\jdk-17"
Push-Location "$dst\android"; .\gradlew.bat assembleDebug; Pop-Location
```

## 주의할 점
- 파일 응답 200만으로는 오프라인 동작을 보장하지 못한다. 실제 기기에서 한 번 연 뒤 비행기 모드로 아이콘을 다시 열어 확인한다.
- 공개 저장소에는 앱 코드만 올리고, 사용자의 음원·문서 같은 데이터나 개인 설정은 올리지 않는다. 배포하는 콘텐츠의 저작권도 확인한다.
- 사용자가 직접 기기에 넣는 파일은 iPhone 파일 앱(카카오톡 "파일에 저장", iCloud Drive)에서 보이는 위치에 있어야 `<input type="file">`로 고를 수 있다.

## 참고
- GitHub Docs: GitHub Pages 사용 한도
- MDN: Progressive web apps, Service Worker API, Web app manifest
- Android Gradle Plugin 릴리스 노트(필요 JDK 버전)
