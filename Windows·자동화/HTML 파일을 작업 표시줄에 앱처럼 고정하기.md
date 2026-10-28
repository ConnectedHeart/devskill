# HTML 파일을 작업 표시줄에 앱처럼 고정하기

> 분류: Windows·자동화 · 최근 갱신: 2026-10-27

## 한 줄 요약
Windows 11은 `.html` 파일을 작업 표시줄에 바로 고정할 수 없으므로, `chrome.exe --app="file:///..."` 바로가기를 만들어 그 바로가기를 고정한다. 주소창 없는 별도 창으로 열리고 아이콘도 따로 잡힌다.

## 내용
- 작업 표시줄 고정은 실행 파일(또는 그 바로가기) 단위다. 문서 파일은 고정 대상이 아니다.
- Chrome(또는 Edge)의 `--app=<URL>` 옵션은 탭·주소창 없이 앱 창으로 연다. 작업 표시줄에서 일반 브라우저 창과 따로 묶인다.
- 바로가기에 `.ico` 파일을 지정하면 그 아이콘이 고정된 버튼에 쓰인다. 고정하기 전에 아이콘을 바꿔 두는 편이 편하다. 여러 크기(16~256px)를 한 `.ico`에 넣으면 해상도마다 선명하다.
- 고정 동작은 Windows가 프로그램에서 하지 못하게 막아 두었다. 바로가기를 만드는 것까지는 스크립트로 되고, 고정은 사용자가 우클릭 → 추가 옵션 표시 → 작업 표시줄에 고정으로 한다.
- 고정 후 아이콘이 기본 브라우저 아이콘으로 보이면 아이콘 캐시 문제다. 로그아웃 후 다시 로그인하면 바뀐다.

## 핵심 코드
### 바로가기 항목 위치
```text
"C:\Program Files\Google\Chrome\Application\chrome.exe" --app="file:///C:/Users/<사용자>/Desktop/viewer.html"
```

### PowerShell로 바로가기(.lnk) 만들기
```powershell
$ws  = New-Object -ComObject WScript.Shell
$lnk = $ws.CreateShortcut("$env:USERPROFILE\Desktop\뷰어.lnk")
$lnk.TargetPath   = "C:\Program Files\Google\Chrome\Application\chrome.exe"
$lnk.Arguments    = '--app="file:///C:/Users/<사용자>/Desktop/viewer.html"'   # 바꿀 곳: HTML 경로
$lnk.IconLocation = "$env:USERPROFILE\Desktop\viewer.ico,0"                    # 바꿀 곳: 아이콘
$lnk.Save()
```

## 주의할 점
- 경로에 공백·한글이 있으면 `file:///` URL 전체를 따옴표로 감싼다(브라우저가 알아서 인코딩한다).
- Chrome 설치 경로가 사용자별(`%LOCALAPPDATA%\Google\Chrome\Application`)일 수 있으니 확인한다.
- `file://` 페이지는 일부 브라우저 기능(fetch로 다른 로컬 파일 읽기 등)이 막힌다.

## 참고
- Chromium 명령줄 스위치 `--app`
