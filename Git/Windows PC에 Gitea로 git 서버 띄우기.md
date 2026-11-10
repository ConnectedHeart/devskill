# Windows PC에 Gitea로 git 서버 띄우기

> 분류: Git · 최근 갱신: 2026-11-09

## 한 줄 요약
같은 망의 몇 사람과 저장소를 공유할 때는 Gitea(exe 하나, 내장 SQLite)를 한 PC에 띄우고 HTTP 포트 하나만 내부 대역에 열면 된다. 다른 사람은 git과 브라우저만 있으면 된다.

## 내용
### 선택지 비교
| 방법 | 특징 |
|---|---|
| 로컬 bare 저장소(`C:/repo/x.git`) | 그 PC에서만 보인다 |
| Windows 폴더 공유(SMB) | 간단하지만 권한 관리가 공유 권한뿐 |
| OpenSSH 서버 | 계정 인증, 설정이 많음 |
| git daemon | 인증 없음, 권하지 않음 |
| **Gitea** / Forgejo | 웹 화면·계정·팀 권한·PR, 메모리 100~200MB |
| GitLab CE | Linux/Docker, 4GB 이상 |

### 설치와 설정 요점
- 공식 배포처에서 exe와 `.sha256`을 받아 해시를 확인하고 `<설치폴더>\gitea.exe`로 둔다. 설정은 `custom\conf\app.ini`, 데이터는 `data\`(DB·저장소), 로그는 `log\`.
- 기본값은 넓게 열려 있다: 모든 주소에서 받고(`0.0.0.0`), 누구나 가입하고, 로그인 없이 공개 저장소를 본다. 처음부터 막아 둔다.
- `ROOT_URL`은 clone 주소와 로그인 후 돌아갈 주소를 만든다. **다른 PC가 실제로 접속하는 주소**여야 한다. PC 이름을 쓰면 상대 PC에서 그 이름이 풀려야 한다(회사 DNS 등록, 아니면 같은 대역 브로드캐스트 이름 해석). 임의 이름은 DNS나 PC마다 hosts가 필요하다.
- SSH를 쓰지 않으면 HTTP 포트 하나만 열면 된다. 인증은 계정 비밀번호이고 Git Credential Manager가 기억한다.
- 설치 화면은 app.ini 값으로 칸을 채우고, 설치 버튼을 누르면 화면 값을 다시 app.ini에 쓴다. SSH를 꺼도 화면의 SSH 포트 칸에 기본값 22가 다시 나오니 비워야 한다. 설치 후 app.ini에는 내부 토큰·JWT 비밀 키가 생기므로 공유하지 않는다.
- 가입을 막았으면 설치 화면에서 관리자 계정을 꼭 만든다(안 만들면 명령줄 `gitea admin user create`로 만들어야 함). 설치를 끝내기 전에 외부에 열면 다른 사람이 관리자를 먼저 만들 수 있다.

### 방화벽
- 프로그램과 포트, 원격 대역을 함께 지정한 허용 규칙을 **먼저** 만든다. 그러면 Windows "네트워크 허용" 창이 뜨지 않는다.
- 그 창에서 취소를 누르면 그 프로그램 **차단 규칙**이 생기고, 차단은 허용보다 우선하므로 허용 규칙을 넣어도 막힌다.
- PC가 손님용 망 등 다른 망에 붙어 있으면 방화벽과 무관하게 내부 PC가 닿지 못한다.

### 공유와 권한
- 관리자 → 사이트 관리 → 사용자 만들기("비밀번호 변경 요구" 체크).
- 조직 → 팀(리포지토리: 모든 리포지토리, 코드: 쓰기)에 사람을 넣는 방식이 편하다. 저장소 하나만 열 때는 저장소 설정의 협업자. 읽기는 clone·pull, 쓰기는 push까지. Owners 팀은 조직 관리자라 일반 개발자는 넣지 않는다.
- 조직·사용자 "공개 범위"는 프로필과 구성원 노출만 정한다. 저장소 접근 권한과는 별개다.

### 겪은 문제
| 증상 | 원인 | 해결 |
|---|---|---|
| push가 404로 끊기고 `Everything up-to-date`가 표시됨 | Gitea를 띄운 셸에 `GIT_CONFIG_PARAMETERS='safe.bareRepository=explicit'`가 있어 서버의 git이 bare 저장소를 거부 | 그 변수를 지우고 Gitea를 띄운다(더블클릭·일반 창이면 해당 없음) |
| 다른 PC의 첫 clone이 멈춤 | 브라우저 OAuth 로그인 중 임시 비밀번호 변경 화면으로 갔다가 홈으로 돌아가, Credential Manager가 승인 응답을 계속 기다림 | Ctrl+C 후 clone 재실행, "애플리케이션 승인"에서 승인 |
| 비밀번호를 바꾼 뒤 계속 실패 | 저장된 자격 증명 | Windows 자격 증명 관리자에서 `git:http://<주소>:<포트>` 삭제 |
| push만 403으로 거부, fetch·pull은 됨. 서버 로그에 로그인 200 뒤 pre-receive 단계 `User permission denied for writing` | 그 사용자에게 저장소 코드 **쓰기** 권한이 없음(팀 권한이 읽기이거나 팀 구성원이 아님). 보호 브랜치에 막힌 경우는 다른 메시지 | 조직 → 팀 편집에서 "일반 접근" + 코드 "쓰기", 구성원 목록 확인. 커밋은 다시 만들 필요 없이 push만 재시도 |
| 가끔 한 번 `Failed to authenticate user`, 바로 다시 하면 성공. 서버 로그 `Failed to verify user: user does not exist [uid: 0, name: OAUTH_USER]` | Credential Manager가 브라우저 로그인으로 받은 OAuth 액세스 토큰(약 1시간)이 만료. 실패한 명령이 저장 토큰을 지우고, 다음 명령이 갱신 토큰으로 새 토큰을 받음 | 한 번 더 실행. 연속으로 실패하면 자격 증명 항목을 지우고 다시 로그인 |

## 핵심 코드
### 1) 받은 파일 확인과 실행
```powershell
certutil -hashfile gitea-<버전>-windows-amd64.exe SHA256     # .sha256 파일 값과 비교
& 'C:\gitea\gitea.exe' web --config 'C:\gitea\custom\conf\app.ini'
```

### 2) app.ini 핵심 값
```ini
[server]
; 처음엔 127.0.0.1로 두고 설치·방화벽 뒤에 0.0.0.0으로
HTTP_ADDR = 0.0.0.0
HTTP_PORT = 3000
ROOT_URL  = http://<PC이름 또는 IP>:3000/
DISABLE_SSH = true
START_SSH_SERVER = false

[service]
DISABLE_REGISTRATION = true
REQUIRE_SIGNIN_VIEW = true

[repository]
DEFAULT_PRIVATE = private
```

### 3) 방화벽: 프로그램·포트·내부 대역만 허용 (관리자 PowerShell)
```powershell
New-NetFirewallRule -DisplayName "Gitea 3000 (internal)" -Direction Inbound `
  -Program "C:\gitea\gitea.exe" -Protocol TCP -LocalPort 3000 -Action Allow `
  -RemoteAddress <내부대역/CIDR>     # 실제 내부 대역으로
Get-NetFirewallApplicationFilter -Program "C:\gitea\gitea.exe" | Get-NetFirewallRule |
  Select-Object DisplayName, Action, Enabled       # 차단 규칙이 없는지 확인
```

### 4) 특정 환경 변수를 빼고 띄우는 시작 스크립트
```powershell
# start-gitea.ps1 — 셸에서 상속된 git 설정 변수 제거 후 실행
Remove-Item Env:GIT_CONFIG_PARAMETERS -ErrorAction SilentlyContinue
& 'C:\gitea\gitea.exe' web --config 'C:\gitea\custom\conf\app.ini'
```

### 5) 기존 저장소 올리기와 다른 PC에서 받기
```bash
# 웹에서 빈 저장소(README 등 초기화 없이)를 먼저 만든 뒤
git remote set-url origin http://localhost:3000/<조직>/<저장소>.git
git push origin --all && git push origin --tags
# 다른 PC
git clone http://<PC이름 또는 IP>:3000/<조직>/<저장소>.git
```

### 6) 이름이 풀리는지 다른 PC에서 확인
```bash
ping <PC이름>      # IP가 표시되면 이름 해석 성공(응답 시간 초과여도 됨)
```

### 7) push 거부·인증 실패 원인을 서버 로그에서 찾기
```powershell
# 쓰기 권한 거부(403)와 토큰 만료 흔적 찾기 - 로그 경로는 설치 위치에 맞게
Select-String -Path 'C:\gitea\log\gitea.log' -Pattern 'permission denied for writing','Failed to verify user' |
  Select-Object -Last 20
```

## 주의할 점
- 서버 PC가 꺼지거나 다른 망으로 옮기면 모두 끊긴다. 디스크 한 곳뿐이라 백업도 따로 둔다. 상시 쓸 저장소라면 늘 켜진 서버에 두는 편이 낫다.
- 고객사 소스 등을 PC 서버로 여는 것이 보안 정책에 맞는지 먼저 확인한다.
- http라 clone 때 "unencrypted HTTP" 경고가 나온다. 사내 한정 용도가 아니면 https(리버스 프록시)를 둔다.
- 서비스 등록 시 `RUN_USER`를 `<PC이름>$`로 바꿔야 한다.

## 참고
- Gitea 문서: Installation from binary, Windows Service, Config Cheat Sheet
