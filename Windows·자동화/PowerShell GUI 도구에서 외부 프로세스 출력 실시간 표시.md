# PowerShell GUI 도구에서 외부 프로세스 출력 실시간 표시

> 분류: Windows·자동화 · 최근 갱신: 2026-10-27

## 한 줄 요약
WinForms 창에서 빌드·ssh 같은 명령을 창 없이 실행하고 출력을 로그 상자에 흘려보내려면, 표준출력을 파이프로 돌려 비동기로 한 줄씩 읽으면서 UI 이벤트를 주기적으로 처리한다.

## 내용
- **창 없이 실행**: `ProcessStartInfo`에 `CreateNoWindow = $true`, `UseShellExecute = $false`를 주고 `RedirectStandardOutput`·`RedirectStandardError`를 켠다. 그러면 콘솔 창이 뜨지 않고 출력이 전부 파이프로 들어온다. `.cmd`로 된 명령(예: Maven의 `mvn`)은 `cmd.exe /c`를 거쳐 실행한다.
- **한 줄씩 읽기**: stdout과 stderr 각각에 `ReadLineAsync()`를 걸고, 완료된 쪽의 줄을 꺼낸 뒤 다음 읽기를 다시 건다. 두 스트림을 따로 읽으므로 둘 사이의 줄 순서는 실제 출력 순서와 조금 어긋날 수 있다.
- **화면이 멈추지 않게**: 별도 스레드를 쓰지 않는 단순한 구조라면, 읽을 줄이 없을 때 `[System.Windows.Forms.Application]::DoEvents()`를 호출하고 잠깐 쉰다. 줄이 쏟아질 때도 일정 횟수마다 `DoEvents()`를 불러야 중지 버튼이 눌린다. 출력이 덩어리로 갱신되어 보이는 것은 이 때문이다.
- **중지**: 자식이 다시 자식을 띄우는 명령(빌드 도구가 띄우는 JVM 등)은 `taskkill /T /F /PID <pid>`로 프로세스 트리째 끝낸다.
- **인코딩**: 읽는 쪽 인코딩을 프로그램에 맞춰야 한다. Windows용 도구는 시스템 기본 코드페이지로, ssh 계열은 UTF-8로 내보내는 경우가 많다.
- **질문이 뜨면 멈춘다**: 창 없이 실행하면 사용자가 답할 방법이 없다. 대화형 질문이 나올 수 있는 명령은 질문 대신 바로 성공하거나 실패하도록 옵션을 준다(ssh의 `BatchMode=yes` 등).
- **종료 상태를 로그 끝에 남기기**: 작업이 끝나면 성공·실패·중지 중 무엇인지와 소요 시간을 마지막 줄에 항상 찍는다. 로그만 보고 끝났는지 알 수 있어야 한다.
- **실행기**: `.bat`에서 `powershell.exe -NoProfile -ExecutionPolicy Bypass -STA -WindowStyle Hidden -File "%~dp0Tool.ps1"`로 띄운다. `-STA`는 WinForms에 필요하고 `%~dp0`은 bat이 있는 폴더라 폴더째 옮겨도 동작한다.

## 핵심 코드
### 1) 실행기 bat (Tool.ps1 옆에 둔다)
```bat
@echo off
rem %~dp0 = 이 bat이 있는 폴더. -STA는 WinForms에 필요
start "" powershell.exe -NoProfile -ExecutionPolicy Bypass -STA -WindowStyle Hidden -File "%~dp0Tool.ps1"
```

### 2) 창, 로그 상자, 실행·중지 버튼
```powershell
Add-Type -AssemblyName System.Windows.Forms, System.Drawing
$projectDir = 'C:\<경로>\project'                       # 명령을 실행할 폴더
$form = New-Object System.Windows.Forms.Form
$form.Text = 'Build Tool'; $form.Width = 900; $form.Height = 600

$logBox = New-Object System.Windows.Forms.TextBox
$logBox.Multiline = $true; $logBox.ReadOnly = $true; $logBox.ScrollBars = 'Vertical'
$logBox.Dock = 'Fill'; $logBox.Font = New-Object System.Drawing.Font('Consolas', 9)

$btnRun  = New-Object System.Windows.Forms.Button; $btnRun.Text  = 'Run';  $btnRun.Dock  = 'Top'
$btnStop = New-Object System.Windows.Forms.Button; $btnStop.Text = 'Stop'; $btnStop.Dock = 'Top'
$form.Controls.AddRange(@($logBox, $btnStop, $btnRun))  # Fill 컨트롤을 맨 앞에 넣어야 남은 영역을 채운다

$script:stopRequested = $false
$btnStop.Add_Click({ $script:stopRequested = $true })
$btnRun.Add_Click({ Invoke-ToolCommand -Command 'mvn clean package' -WorkDir $projectDir })
# Invoke-ToolCommand 는 3)의 함수. ShowDialog 보다 위에 정의한다
[void]$form.ShowDialog()
```

### 3) stdout·stderr 함께 읽기, 중지, 종료 상태 남기기
```powershell
function Invoke-ToolCommand([string]$Command, [string]$WorkDir) {
    $si = New-Object System.Diagnostics.ProcessStartInfo
    $si.FileName = 'cmd.exe'; $si.Arguments = "/c $Command"; $si.WorkingDirectory = $WorkDir
    $si.UseShellExecute = $false; $si.CreateNoWindow = $true
    $si.RedirectStandardOutput = $true; $si.RedirectStandardError = $true
    $si.StandardOutputEncoding = [Text.Encoding]::Default   # 5.1에서는 시스템 코드페이지. ssh 등은 [Text.Encoding]::UTF8
    $si.StandardErrorEncoding  = [Text.Encoding]::Default
    $sw = [Diagnostics.Stopwatch]::StartNew(); $script:stopRequested = $false; $status = $null
    $p = [System.Diagnostics.Process]::Start($si)
    $out = $p.StandardOutput.ReadLineAsync(); $err = $p.StandardError.ReadLineAsync(); $n = 0
    while ($out -or $err) {
        $got = $false
        if ($out -and $out.IsCompleted) {
            if ($null -eq $out.Result) { $out = $null } else { $logBox.AppendText($out.Result + "`r`n"); $out = $p.StandardOutput.ReadLineAsync(); $got = $true }
        }
        if ($err -and $err.IsCompleted) {
            if ($null -eq $err.Result) { $err = $null } else { $logBox.AppendText('[ERR] ' + $err.Result + "`r`n"); $err = $p.StandardError.ReadLineAsync(); $got = $true }
        }
        if ($script:stopRequested) { taskkill /T /F /PID $p.Id | Out-Null; $status = 'STOPPED'; break }  # 자식 프로세스까지
        if (-not $got) { [System.Windows.Forms.Application]::DoEvents(); Start-Sleep -Milliseconds 20 }
        elseif ((++$n % 50) -eq 0) { [System.Windows.Forms.Application]::DoEvents() }   # 줄이 쏟아질 때도 버튼이 눌리게
    }
    $p.WaitForExit()
    if (-not $status) { $status = if ($p.ExitCode -eq 0) { 'SUCCESS' } else { "FAILED (exit $($p.ExitCode))" } }
    $logBox.AppendText("==== $status, elapsed $($sw.Elapsed.ToString('hh\:mm\:ss')) ====`r`n")
}
```
ssh처럼 질문이 뜰 수 있는 명령은 질문 대신 실패하도록 준다: `ssh -o BatchMode=yes -o ConnectTimeout=10 <사용자>@<호스트> "<명령>"`.

### 4) stdout 한 줄씩 읽기 (가장 단순한 형태)
```powershell
$si = New-Object System.Diagnostics.ProcessStartInfo
$si.FileName = 'cmd.exe'
$si.Arguments = '/c mvn clean package'
$si.WorkingDirectory = $projectDir
$si.UseShellExecute = $false
$si.CreateNoWindow = $true
$si.RedirectStandardOutput = $true
$si.RedirectStandardError = $true

$p = [System.Diagnostics.Process]::Start($si)
$task = $p.StandardOutput.ReadLineAsync()
while ($true) {
    if ($task.IsCompleted) {
        $line = $task.Result
        if ($null -eq $line) { break }          # 스트림 끝
        $logBox.AppendText($line + "`r`n")
        $task = $p.StandardOutput.ReadLineAsync()
    } else {
        [System.Windows.Forms.Application]::DoEvents()
        Start-Sleep -Milliseconds 20
    }
}
$p.WaitForExit()
$p.ExitCode
```

### 5) 주석 줄이 있는 JSON 설정 읽기
```powershell
# settings.json 안에서 // 로 시작하는 줄만 주석으로 걸러 낸다(값 뒤 주석은 안 됨)
$path = Join-Path $PSScriptRoot 'settings.json'
$json = (Get-Content $path -Encoding UTF8 | Where-Object { $_ -notmatch '^\s*//' }) -join "`n"
$config = $json | ConvertFrom-Json
```

## 주의할 점
- 스크립트를 exe로 감싸도 소스는 숨겨지지 않는다. 실행기 exe는 스크립트가 옆에 그대로 있고, 스크립트를 exe 안에 묶는 방식은 공개된 방법으로 원문을 꺼낼 수 있으며, .NET으로 다시 짜도 디컴파일된다. 숨겨야 할 것이 로직이라면 로직을 서버 쪽에 두어야 한다.
- 실제로 민감한 것은 대개 소스가 아니라 설정 파일(서버 주소·계정)과 저장된 로그다. 도구를 넘길 때는 예시 값만 넣은 설정 파일을 주고 로그·이력 폴더는 뺀다.
- 스크립트를 exe 하나로 묶는 도구는 백신 오탐이 흔하고, 스크립트를 고칠 때마다 다시 빌드해야 한다.
- JSON은 주석을 지원하지 않는다. 설정 파일에 주석이 필요하면 읽는 쪽에서 특정 접두어로 시작하는 줄을 걸러낸 뒤 파싱한다(값 뒤에 붙는 주석은 처리할 수 없다).

## 참고
- .NET 문서: System.Diagnostics.ProcessStartInfo, Process.StandardOutput
