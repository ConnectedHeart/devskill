# 훅(hook) 작성 요령

> 분류: Claude Code · 최근 갱신: 2026-10-27

## 한 줄 요약
훅은 질문·답변·세션 시작마다 실행되므로 "할 일이 없을 때 얼마나 빨리 끝나는가"가 가장 중요하고, 훅의 출력은 Claude의 컨텍스트로 들어간다.

## 내용
### 훅이 도는 시점
- `SessionStart`: 세션이 시작될 때. `matcher`로 `startup` 등 시작 종류를 고를 수 있다.
- `UserPromptSubmit`: 질문을 보낼 때마다
- `Stop`: 답변이 끝날 때마다

- `SubagentStart` / `SubagentStop`: 서브에이전트가 시작·종료될 때. 훅 입력 JSON에 `agent_id`가 들어온다.

질문마다 도는 훅이 40초를 기다리면 질문마다 40초가 늦어진다. 훅 안의 재시도와 대기 시간은 최악의 경우 그대로 체감 지연이 된다.

### 빠르게 끝내기
- **가벼운 사전 검사를 먼저**: Windows에서 PowerShell은 기동과 스크립트 로드에만 0.3초가량 든다. Git Bash가 있으면 훅 명령은 bash로 실행되므로, bash 한 줄로 조건을 먼저 보고 필요할 때만 PowerShell을 띄운다.
- **표시 파일로 하루 한 번만**: "오늘 이미 확인함" 파일을 남기고, 훅 명령 맨 앞에서 `test -f`로 확인해 있으면 바로 끝낸다.
- **조건이 안 맞으면 즉시 종료**: 특정 환경에서만 의미 있는 훅은 그 환경이 아닐 때 아무것도 하지 않고 끝낸다.
- 측정해 보면 원인이 예상과 다른 경우가 많다. 네트워크 확인이 느릴 줄 알았지만 실제로는 셸 기동 시간이 대부분이었다.

### 훅 출력은 Claude에게 전달된다
- `SessionStart`, `UserPromptSubmit` 훅의 표준 출력은 컨텍스트에 추가된다. 할 일이 없으면 **아무것도 출력하지 않는다**.
- Claude에게 무언가 시키려면 "첫 답변에서 이것을 물어보라"처럼 구체적인 지시문을 출력한다.
- Windows에서는 훅 출력의 한글이 깨질 수 있다. 지시문은 영어로 출력하고 "한국어로 답하라"고 적는 편이 안전하다.

### Stop 훅과 백그라운드 서브에이전트
- 서브에이전트는 같은 세션 안에서 돌기 때문에, 메인 답변이 끝나면 Stop 훅이 서브에이전트가 아직 돌고 있어도 실행된다. Stop 훅이 네트워크를 바꾸거나 자원을 정리하는 일을 하면 백그라운드 서브에이전트가 그 순간 API 연결을 잃고 실패한다(대기했다가 이어지지 않는다).
- 이를 피하려고 메인 답변에서 sleep으로 서브에이전트를 기다리면 화면에서는 몇 분씩 "처리 중"으로 보인다. 근본 해결은 `SubagentStart`에서 "작업 중" 표시 파일을 만들고 Stop 훅이 그 표시가 있으면 정리를 미루게 하는 것이다.
- `SubagentStop`에서는 표시만 지우고 정리는 하지 않는다. 백그라운드 서브에이전트가 끝나면 메인 세션이 결과를 받으려고 다시 답변하므로, 정리는 그 후속 답변의 Stop 훅이 하게 둔다.
- 비정상 종료로 표시가 남을 수 있으니 일정 시간(예: 60분)이 지난 표시는 무시한다.
- 질문 창(사용자 선택 대기)을 띄운 채 답이 없으면 Stop 훅이 돌지 않아 "답변 중" 표시가 계속 남는다. 여러 세션이 표시를 공유하는 구조라면 다른 세션의 정리도 그만큼 밀린다.
- 무거운 점검(전체 문서 검사 등)은 매번 돌리지 말고, 바뀐 파일만 대상으로 좁히거나 매일 바뀌는 파일을 대상에서 빼면 실제 시간이 가장 많이 준다.

### 설정에 넣는 명령
- JSON 안에 긴 bash 한 줄을 직접 넣으면 따옴표와 역슬래시 이스케이프 때문에 다루기 어렵다. 스크립트 파일로 빼고 설정에는 실행 명령만 둔다.
- `timeout`을 지정해 훅이 멈춰도 세션이 무한정 기다리지 않게 한다.

## 핵심 코드
### 1) 설정: 시작·질문마다 도는 훅 (~/.claude/settings.json)
```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [
          {
            "type": "command",
            "command": "test -f ~/.claude/mycheck/checked-$(date +%F) || powershell -NoProfile -ExecutionPolicy Bypass -File ~/.claude/mycheck/check.ps1",
            "timeout": 30
          }
        ]
      }
    ],
    "UserPromptSubmit": [
      {
        "hooks": [
          { "type": "command", "command": "bash ~/.claude/mycheck/pre.sh", "timeout": 60 }
        ]
      }
    ]
  }
}
```

### 2) 빨리 끝나는 훅 스크립트 (pre.sh)
```bash
#!/usr/bin/env bash
# 훅 입력 JSON 은 표준 입력으로 들어온다. 할 일이 없으면 아무것도 출력하지 않고 끝낸다
input=$(cat)
flag=~/.claude/mycheck/checked-$(date +%F)
[ -f "$flag" ] && exit 0                               # 오늘 이미 확인함
[ -d "<조건 폴더>" ] || exit 0                          # 해당 환경이 아니면 즉시 종료
# 필요할 때만 PowerShell 을 띄운다
result=$(powershell -NoProfile -ExecutionPolicy Bypass -File ~/.claude/mycheck/check.ps1)
[ -z "$result" ] && { touch "$flag"; exit 0; }
# 출력은 Claude 컨텍스트로 들어간다. 영어로 쓰고 답변 언어를 지정한다
echo "Daily check found: $result. Ask the user about it in your first reply, in Korean."
touch "$flag"
```

### 3) 서브에이전트가 도는 동안 Stop 정리를 미루는 훅
```json
{
  "hooks": {
    "SubagentStart": [ { "hooks": [ { "type": "command", "command": "bash ~/.claude/mycheck/sub.sh start", "timeout": 10 } ] } ],
    "SubagentStop":  [ { "hooks": [ { "type": "command", "command": "bash ~/.claude/mycheck/sub.sh stop",  "timeout": 10 } ] } ],
    "Stop":          [ { "hooks": [ { "type": "command", "command": "bash ~/.claude/mycheck/on-stop.sh", "timeout": 30 } ] } ]
  }
}
```
```bash
#!/usr/bin/env bash
# ~/.claude/mycheck/sub.sh start|stop — 입력 JSON 의 agent_id 로 표시 파일을 만들고 지운다
dir=~/.claude/mycheck/active; mkdir -p "$dir"
id=$(cat | grep -oE '"agent_id"[[:space:]]*:[[:space:]]*"[^"]+"' | sed -E 's/.*"([^"]+)"$/\1/')
[ -z "$id" ] && exit 0
case "$1" in
  start) touch "$dir/$id" ;;
  stop)  rm -f "$dir/$id" ;;        # 표시만 지우고 정리는 하지 않는다
esac
```
```bash
#!/usr/bin/env bash
# ~/.claude/mycheck/on-stop.sh — 60분 안의 표시가 있으면 정리를 미룬다
dir=~/.claude/mycheck/active
find "$dir" -type f -mmin +60 -delete 2>/dev/null     # 비정상 종료로 남은 오래된 표시 무시
[ -n "$(ls -A "$dir" 2>/dev/null)" ] && exit 0         # 서브에이전트가 아직 돌고 있음
bash ~/.claude/mycheck/cleanup.sh                      # 실제 정리 작업
```

### 4) 조건별로 직접 실행해 출력과 시간 확인
```bash
rm -f ~/.claude/mycheck/checked-$(date +%F)
time (echo '{"prompt":"hi"}' | bash ~/.claude/mycheck/pre.sh)    # 첫 실행: 지시문이 나오는지
time (echo '{"prompt":"hi"}' | bash ~/.claude/mycheck/pre.sh)    # 두 번째: 출력 없이 바로 끝나는지
echo '{"agent_id":"test1"}' | bash ~/.claude/mycheck/sub.sh start && ls ~/.claude/mycheck/active
echo '{"agent_id":"test1"}' | bash ~/.claude/mycheck/sub.sh stop  && ls ~/.claude/mycheck/active
```

## 주의할 점
- 상태를 표시 파일로 관리하면 훅이 중간에 실패했을 때 파일이 남는다. 답변이 끝날 때 도는 훅에서 남은 표시 파일을 정리한다.
- 로그 문구는 실제 판단 근거를 적는다. 추정한 상태를 적으면 다른 환경에서 틀린 로그가 된다.
- 예약 작업으로 뜨는 무인 세션도 훅을 탄다. "그날 첫 세션" 같은 판단에서 무인 세션을 세지 않도록 시간대를 제외하는 등의 처리가 필요하다.
- 스크립트를 고친 뒤에는 조건별로(해당 없음, 첫 실행, 두 번째 실행) 직접 실행해 출력과 소요 시간을 확인한다.

## 참고
- Claude Code 문서: Hooks
