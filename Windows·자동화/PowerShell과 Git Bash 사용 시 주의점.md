# PowerShell과 Git Bash 사용 시 주의점

> 분류: Windows·자동화 · 최근 갱신: 2026-10-03

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

## 예시
```powershell
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
