# PowerShell 기초 문법(Java 개발자용)

> 분류: Windows·자동화 · 최근 갱신: 2026-10-08

## 한 줄 요약
PowerShell은 Windows에 기본 포함된 .NET 기반 셸 겸 스크립트 언어로, 변수는 `$이름`, 비교는 `-eq` 같은 단어 연산자, 파이프는 글자가 아니라 객체를 넘긴다.

## 내용
- **성격**: 컴파일 없이 `powershell.exe`가 `.ps1` 파일을 읽어 실행한다. .NET 라이브러리를 그대로 쓸 수 있어 설치 없이 창(WinForms)까지 만들 수 있다. `.ps1`은 더블클릭하면 메모장으로 열리는 것이 기본이라, 실행용 `.bat`을 따로 두는 경우가 많다.
- **변수와 스코프**: 타입 선언과 세미콜론이 없다. `$script:이름`은 스크립트 전체가 공유하고(Java의 필드), 접두어가 없으면 함수 안에서만 산다. 한 함수에서 넣고 다른 함수에서 읽을 값에 `$script:`를 빼먹으면 함수가 끝날 때 사라진다.
- **미리 정해진 변수**: `$true`, `$false`, `$null`, 그리고 `$_`(파이프나 `catch` 안에서 지금 처리 중인 것). `$MyInvocation.MyCommand.Path`는 실행 중인 스크립트의 전체 경로다.
- **문자열**: 큰따옴표는 변수를 값으로 바꾸고 작은따옴표는 글자 그대로 둔다. 속성은 `$( )`로 감싸고, 줄바꿈은 `\n`이 아니라 백틱이다.
- **연산자**: `==`/`!=` 대신 `-eq`/`-ne`, 대소 비교는 `-gt`/`-lt`/`-ge`/`-le`, 논리는 `-and`/`-or`/`-not`. 문자열용으로 `-match`(정규식), `-replace`, `-split`, `-join`이 있다.
- **`@` 기호**: `@( )`는 배열, `@{ }`는 해시테이블(Java의 Map), `@' … '@`는 여러 줄 문자열이다.
- **값 자체가 조건**: `if` 안에 아무 값이나 넣을 수 있고 `$null`, 빈 문자열, 빈 배열, 0은 거짓이다.
- **파이프**: `|` 왼쪽의 결과 객체가 오른쪽 명령의 입력으로 넘어간다. 배열이면 항목이 하나씩 넘어가고 받는 쪽 `{ }` 안에서 `$_`로 받는다. Java 스트림의 `map`과 같은 구조다.
- **cmdlet과 함수**: 기본 제공 명령을 cmdlet이라 부르고 이름은 동사-명사 규칙이다. `function`으로 만든 함수도 호출 모양이 같다. 괄호와 쉼표 없이 `이름 인자1 -옵션 값`으로 부르고, 값 없는 `-옵션`은 스위치(켜기)다.
- **경로 명령**: `Split-Path -Parent`(상위 폴더), `Split-Path -Leaf`(마지막 이름), `Join-Path`(이어 붙이기), `Test-Path`(존재 확인). 글자만 다루는 명령이라 실제 파일이 있는지는 `Test-Path`로 따로 본다.
- **예외**: `throw "메시지"`로 던지고 `try { } catch { $_.Exception.Message }`로 받는다.

## 예시
```powershell
# 스크립트가 있는 폴더 기준으로 설정 파일 찾기 (폴더를 옮겨도 동작)
$script:BaseDir = Split-Path -Parent $MyInvocation.MyCommand.Path
$configPath = Join-Path $script:BaseDir 'settings.json'

# JSON 을 객체로 읽기
$config = Get-Content -Path $configPath -Raw -Encoding UTF8 | ConvertFrom-Json
if (-not $config.servers) { throw "servers 항목이 없습니다." }

# 항목이 1개여도 배열로 다루기
foreach ($server in @($config.servers)) {
    "이름: $($server.name), 포트: $($server.port)"
}

# 최신 파일 하나 고르기 (객체가 넘어가므로 속성으로 정렬 가능)
Get-ChildItem -Filter '*.war' | Sort-Object LastWriteTime -Descending | Select-Object -First 1
```

| Java | PowerShell |
|---|---|
| `new File(p).getParent()` | `Split-Path -Parent $p` |
| `list.stream().map(s -> s.name)` | `$list \| ForEach-Object { $_.name }` |
| `a == b && !c` | `$a -eq $b -and -not $c` |
| `new Foo()` | `New-Object Foo` |

## 주의할 점
- `Get-Content`는 기본이 줄 단위 배열이다. 파일 전체를 문자열 하나로 받으려면 `-Raw`를 붙인다.
- JSON 배열에 항목이 1개면 배열이 아닌 객체 하나로 풀린다. 인덱스로 꺼내기 전에 `@( )`로 감싼다.
- 함수 인자 이름이 다른 함수의 지역 변수와 같아도 서로 다른 변수다.
- 배치 파일의 `@echo off`에 있는 `@`는 "이 줄을 출력하지 말라"는 뜻으로 PowerShell의 `@`와 무관하다.

## 참고
- Microsoft Learn: about_Scopes, about_Operators, about_Pipelines, about_Quoting_Rules
