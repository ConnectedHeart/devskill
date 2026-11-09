# Claude 데스크톱 앱 메모리 사용량과 세션 보관

> 분류: Claude Code · 최근 갱신: 2026-11-06

## 한 줄 요약
Claude 데스크톱 앱(Code 탭)은 앱을 켠 뒤 열었거나 대화한 세션마다 Claude Code 프로세스(약 0.5GB)를 띄워 두므로, 메모리를 아끼려면 끝난 세션을 보관(Archive)하고 동시에 여는 세션 수를 줄인다. 오래 안 연 세션은 보관하지 않아도 메모리를 쓰지 않는다.

## 내용
### 메모리를 쓰는 곳(16GB PC에서 잰 예)
| 항목 | 크기 | 비고 |
|---|---|---|
| 세션 프로세스 | 세션당 470~620MB | 쉬고 있어도 보관 전까지 떠 있음 |
| 앱 화면(렌더러) | 0.9~1.1GB | 긴 대화를 그릴수록 커짐 |
| 앱 본체·GPU·터미널 호스트 | 약 1.1GB | 기본 |
| 터미널 탭 셸 | 탭당 약 60MB | |
| 세션이 띄운 서버·DB | 수 GB | WAS 힙 상한, DB 버퍼 풀 등 |

### 보관(Archive)
- 세션이 사이드바 목록에서 내려가고, 그 세션의 프로세스와 **터미널 탭이 종료**된다. 탭 안에서 돌던 서버·스크립트도 같이 꺼진다.
- 대화 기록은 남는다. 보관함에서 다시 열고 메시지를 보내면 이어진다. 파일·git 변경은 건드리지 않는다. 삭제와 달리 되돌릴 수 있다.
- 보관함 위치: 사이드바가 목록형이면 상단 필터에서 Archived, 섹션형이면 메뉴의 "모두 보기"로 여는 Sessions 페이지에서 상태 필터를 보관됨으로.
- 대기 중인 메시지나 원격 제어 연결이 있는 세션은 도구로 보관이 거부될 수 있다(사용자가 사이드바에서 직접 보관하면 된다).

### 줄이는 순서
1. 끝난 세션 보관(세션당 약 0.5GB). 서버를 돌리는 세션인지 먼저 본다.
2. 동시에 여는 세션을 2~3개로.
3. 세션이 띄운 개발 서버를 다 쓰면 끄고, 힙 상한(`-Xmx`)을 필요한 만큼만.
4. 긴 대화는 새 세션으로 이어 간다(렌더러·세션 메모리 감소).
5. 안 쓰는 터미널 탭, 상주 앱, 브라우저 탭 정리.

## 핵심 코드
### 1) 프로세스별 메모리 합계 보기 (PowerShell)
```powershell
Get-Process | Group-Object ProcessName |
  Select-Object Name, Count, @{n='MB';e={[int](($_.Group | Measure-Object WorkingSet64 -Sum).Sum/1MB)}} |
  Sort-Object MB -Descending | Select-Object -First 15
```

### 2) 전체 사용량과 커밋 크기
```powershell
$os = Get-CimInstance Win32_OperatingSystem
"free {0:N1} GB / total {1:N1} GB" -f ($os.FreePhysicalMemory/1MB), ($os.TotalVisibleMemorySize/1MB)
(Get-Counter '\Memory\Committed Bytes').CounterSamples.CookedValue / 1GB   # 페이지 파일 포함 사용량
```

### 3) JVM 충돌 기록으로 메모리 부족 확인
```powershell
Get-ChildItem <프로젝트 폴더> -Filter 'hs_err_pid*.log' | Select-String 'insufficient memory' -List
```

## 주의할 점
- 보관은 그 세션의 터미널 탭 프로세스를 끈다. 다른 세션이 쓰고 있는 서버가 그 탭에서 떠 있으면 같이 꺼진다.
- 목록을 정리할 목적이 아니라면 오래된 세션을 한꺼번에 보관할 필요는 없다(프로세스가 없어 메모리 효과가 없다).

## 참고
- Claude Code 문서: Desktop app
