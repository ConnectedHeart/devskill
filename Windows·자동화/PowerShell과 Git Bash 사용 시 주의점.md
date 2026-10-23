# PowerShell과 Git Bash 사용 시 주의점

> 분류: Windows·자동화 · 최근 갱신: 2026-10-21

## 한 줄 요약
Windows에서 스크립트를 짤 때 자주 걸리는 것들 — `curl` 별칭, 줄 끝(CRLF) 변환, 스크립트 파일 인코딩, 결과가 하나일 때의 배열 처리.

## 내용
### curl 은 curl 이 아니다
- Windows PowerShell에서 `curl`은 `Invoke-WebRequest`의 별칭이다. 옵션이 전혀 달라 리눅스용 명령을 그대로 붙이면 실패한다.
- 진짜 curl은 `curl.exe`로 부른다. PowerShell 방식으로 하려면 `Invoke-RestMethod`를 쓴다.
- bash의 `read -s` 같은 비밀값 입력은 `Read-Host -AsSecureString`으로 대신한다.

### sed -i / awk 는 CRLF 를 LF 로 바꾼다
- Git Bash의 `sed -i`, awk로 파일을 고치면 줄 끝이 CRLF에서 LF로 바뀐다. 줄 끝에 민감한 설정 파일은 통째로 변경된 것으로 잡히거나 깨진다.
- CRLF 파일은 PowerShell에서 문자열만 치환해 같은 인코딩으로 다시 쓰는 편이 안전하다.
- git에서 `LF will be replaced by CRLF` 경고는 `core.autocrlf` 설정에 따른 변환 안내이고 오류가 아니다.

### 스크립트 파일 인코딩 (Windows PowerShell 5.1)
- 5.1은 BOM 없는 `.ps1`을 시스템 ANSI 코드페이지(한국어 Windows는 CP949)로 읽는다. UTF-8로 저장한 스크립트 안의 한글 문자열과 경로가 깨진다.
- 한글이 들어간 스크립트는 **UTF-8 with BOM**으로 저장한다.
- 스크립트 출력이 다른 프로그램으로 넘어갈 때도 콘솔 코드페이지 때문에 한글이 깨질 수 있다. 다른 프로그램이 읽을 출력은 영어로 쓰면 문제가 없다.
- `Set-Content`/`Add-Content`는 기본이 ANSI다. 다른 도구가 읽을 파일은 `-Encoding utf8`을 명시한다.

### 결과가 하나면 배열이 아니다
- 파이프라인 결과가 1건이면 배열이 아니라 그 값 자체가 된다. 문자열 1건에 `[-1]`을 쓰면 마지막 항목이 아니라 **마지막 글자**가 나온다.
- 개수와 상관없이 배열로 다루려면 `@(...)`로 감싼다.

### 네이티브 명령의 종료 코드와 stderr
- 네이티브 명령의 성공 여부는 `$?`보다 `$LASTEXITCODE`로 판단한다.
- 5.1에서 네이티브 명령에 `2>&1`을 붙이면 stderr 줄이 오류 객체로 감싸져 명령이 성공해도 실패처럼 보인다.
- 5.1에는 `&&`, `||`가 없다. `A; if ($?) { B }` 형태로 쓴다.

### 실행 정책과 환경 변수
- 터미널에서 `.ps1`이 "이 시스템에서 스크립트를 실행할 수 없으므로…"로 막히면 그 창에서만 `Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force`를 쓴다. 창을 닫으면 원래대로 돌아가고 시스템 설정은 바뀌지 않는다. 일회 실행은 `powershell -ExecutionPolicy Bypass -File x.ps1`도 된다.
- **사용자 환경 변수는 모든 자식 프로세스에 들어간다**: 설정 창에서 등록한 사용자 환경 변수는 새로 뜨는 프로세스가 모두 물려받는다. "변수가 없으면 건너뜀"으로 안전장치를 만들었다면 그 변수가 등록돼 있는 한 안전장치가 아니다.
- 같은 창에서 `$env:X = ...`로 넣은 값은 그 창이 닫힐 때까지 남아, 뒤에 실행하는 스크립트가 등록된 값 대신 그 값을 먼저 쓸 수 있다. 다시 실행하기 전에 `Remove-Item Env:X`로 지운다.

### 함수 이름이 기본 별칭과 겹침
- 스크립트 안에서 만든 짧은 함수 이름(`H`, `R`, `gc` 등)이 기본 별칭(`h`=Get-History, `r`=Invoke-History, `gc`=Get-Content)과 겹치면 예상과 다른 명령이 불린다. 함수 이름은 동사-명사 형태로 길게 짓는다.

### properties의 \uXXXX 이스케이프가 깨지는 경우
- Java `.properties`의 한글은 `가` 같은 이스케이프로 저장되는 경우가 많다. GNU sed의 치환 문자열에서 `\u`는 "다음 글자를 대문자로"라는 뜻이라, 이스케이프를 그대로 넣으려 하면 값이 깨진다. 일부 편집 도구는 이스케이프를 실제 한글로 바꿔 저장하기도 한다.
- 고친 뒤 파일에 비ASCII 바이트가 0인지 확인하고, 이스케이프 형태로 다시 넣는다.

### 폴더 이동이 막힐 때
- 파일 잠금이 하나도 없는데 폴더 이동이 거부되면, IDE(IntelliJ 등)가 그 폴더 안 파일을 연 뒤 폴더 자체를 감시하고 있는 경우가 있다. 탐색기 창이 그 폴더를 열고 있어도 막힌다. 프로그램을 닫기 어렵다면 새 위치로 복사하고 이름·크기를 대조한 뒤 원본 내용을 지운다(빈 폴더는 나중에 지운다).

## 예시
```powershell
# 이 창에서만 실행 정책 우회
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force

# 사용자 환경 변수 값을 명시적으로 읽기
$id = [Environment]::GetEnvironmentVariable("MY_ID", "User")

# 진짜 curl
curl.exe -s -o NUL -w "%{http_code}" https://example.com/api

# 비밀값 입력 후 호출
$sec = Read-Host -AsSecureString "password"
$plain = [Runtime.InteropServices.Marshal]::PtrToStringAuto(
           [Runtime.InteropServices.Marshal]::SecureStringToBSTR($sec))
Invoke-RestMethod -Method Post -Uri https://example.com/token -Body @{ password = $plain }

# 파일을 UTF-8 with BOM 으로 다시 저장
$text = [IO.File]::ReadAllText($path, [Text.Encoding]::UTF8)
[IO.File]::WriteAllText($path, $text, (New-Object Text.UTF8Encoding($true)))

# 1건이어도 배열로
$last = @($items | Sort-Object)[-1]

# CRLF 를 유지한 문자열 치환
$t = [IO.File]::ReadAllText($path)
[IO.File]::WriteAllText($path, $t.Replace('old', 'new'))
```

## 주의할 점
- 파일 첫 3바이트가 `239,187,191`이면 UTF-8 BOM이 있는 것이다: `[IO.File]::ReadAllBytes($path)[0..2]`.
- JSON 설정 안에 bash 한 줄을 직접 넣으면 따옴표와 역슬래시가 이중으로 이스케이프되어 다루기 어렵다. 별도 스크립트 파일로 빼고 설정에는 그 파일 실행 명령만 적는다.
- PowerShell 기동 자체에 0.3초 정도 걸린다. 자주 호출되는 자리에서는 가벼운 사전 검사를 bash로 먼저 하고 필요할 때만 PowerShell을 띄운다.

## 참고
- Microsoft 문서: about_Character_Encoding, about_Arrays, about_Aliases
