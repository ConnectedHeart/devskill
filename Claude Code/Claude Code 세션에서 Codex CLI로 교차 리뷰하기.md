# Claude Code 세션에서 Codex CLI로 교차 리뷰하기

> 분류: Claude Code · 최근 갱신: 2026-11-04

## 한 줄 요약
"Claude가 구현 → Codex가 읽기 전용으로 리뷰 → Claude가 판정·수정"을 바깥 스크립트로 돌리기보다, Claude Code 세션 안에서 `codex exec`를 명령 하나로 부르는 스킬로 만들면 맥락이 유지되고 훅·권한 문제도 줄어든다. 리뷰에는 고른 diff만 보내고 비밀 파일은 입력 단계에서 뺀다.

## 내용
### 바깥 스크립트 방식의 문제
- `claude -p`를 번갈아 부르면 매번 새 세션이라 앞 판단 근거가 없고, 헤드리스라 빌드 명령 권한이 거부되기 쉽다.
- `claude -p`도 사용자 훅을 실행한다. Stop 훅이 네트워크·환경을 바꾸는 PC라면 Claude 단계가 끝날 때마다 바뀌어 다음 Codex 단계가 실패할 수 있다. 세션 안에서 한 답변 동안 부르면 Stop 훅이 돌지 않는다.
- 리뷰 프롬프트에 "git diff를 보라"고 시키면 작업 트리의 비밀 설정 파일 diff까지 외부로 나갈 수 있다. Claude Code의 deny 규칙은 Codex에 적용되지 않고, `--sandbox read-only`는 쓰기만 막고 읽기는 막지 않는다.
- 훅만으로는 "결과를 받아 판단하고 고치는 반복"을 할 수 없다. 이 판단은 구현한 에이전트가 해야 한다.

### 세션 안에서 부르는 절차
1. 변경 경로를 골라(비밀 설정·패치·산출물 제외) diff와 작업 맥락(요청, 변경 의도, 검증 결과, 직전 회차 판정표)을 입력 파일로 만든다. 추가된 줄에 비밀값 같은 패턴이 있으면 값을 출력하지 않고 멈춘다.
2. `codex exec --sandbox read-only`로 입력을 stdin으로 넘기고 마지막 메시지를 파일로 받는다. 출력 첫 줄을 `RESULT: PASS|ISSUES`로 정해 두면 판정이 쉽다.
3. 지적마다 실제 코드로 확인해 반영 / 반영 안 함(이유) / 사람 확인으로 나눈다. 반영했으면 다시 빌드하고 다시 리뷰한다. 최대 2~3회.
4. 결과 보고에 지적별 처리표를 붙인다.

### 사용자가 과정을 볼 수 있게
- 입력·결과 파일을 임시 폴더가 아니라 작업 폴더 안의 무시(gitignore) 폴더에 둔다. 샌드박스 도구가 쓴 임시 경로는 실제 디스크에 없을 수 있다.
- 실행 전 넣은/뺀 파일 목록과 맥락 문서를 채팅에 보여 주고, Codex는 사용자 터미널 탭에서 실행해 진행이 실시간으로 보이게 한다.
- 작업 트리에 다른 작업이 섞여 있으면 임시 git 저장소에 대상 파일만 HEAD 버전으로 커밋한 뒤 작업본을 덮어써, 그 저장소에서 diff를 만든다(변경량이 원본과 같은지 확인).

### Windows 설치에서 막히는 지점
- PowerShell에서 `npm`이 "npm.ps1을 로드할 수 없습니다"로 실패하면 실행 정책 문제다. 정책을 바꾸지 않고 `npm.cmd`, `codex.cmd`로 실행한다.
- 설치 전에 연 터미널은 옛 PATH를 쓰므로 새로 연다.
- 에이전트의 셸 도구가 샌드박스면 `npm install -g`가 도구 안에서는 성공해도 실제 디스크에는 남지 않을 수 있다. 전역 설치는 사용자 터미널에서 한다.
- 로그인(`codex login`)은 사용자가 직접 한다.

## 핵심 코드
### 1) 리뷰 입력 만들기 (Git Bash)
```bash
# 바뀐 소스만, 비밀 설정·산출물 제외
files=$(git status --short | awk '{print $2}' \
  | grep -E '\.(java|xml|jsp|js|sql)$' \
  | grep -v -E '(^|/)(secrets|props)/|application.*\.properties$|/target/')
{
  cat review-prompt.md          # 고정 지시문 + 출력 형식
  echo; cat context.md          # 이번 작업 맥락
  echo; git diff HEAD -- $files
} > review-input.md
# 추가된 줄에 비밀값 패턴이 있으면 중단 (값은 출력하지 않음)
if grep -E '^\+.*(password|apiKey|secret)\s*[=:]' review-input.md >/dev/null; then exit 2; fi
```

### 2) Codex 실행
```bash
codex exec --sandbox read-only --skip-git-repo-check -C "$OUT_DIR" \
  -o "$OUT_DIR/review-1.md" - < "$OUT_DIR/review-input.md"
head -1 "$OUT_DIR/review-1.md"    # RESULT: PASS 또는 RESULT: ISSUES
```

### 3) PowerShell에서 파이프로 넘길 때 (5.1)
```powershell
$OutputEncoding = [Text.UTF8Encoding]::new($false)   # 기본 ASCII라 한글이 ?로 바뀜
Get-Content .\review-input.md -Raw -Encoding UTF8 | codex.cmd exec --sandbox read-only -o review-1.md -
```

## 주의할 점
- 회사 소스를 두 번째 AI 서비스로 보내도 되는지 먼저 확인한다.
- Codex가 작업 폴더의 다른 파일도 읽을 수 있으므로 작업 폴더(`-C`)를 입력 파일만 있는 폴더로 둔다.
- 리뷰 PASS가 실행 검증을 대신하지 않는다. 실행해 보지 못한 DB·브라우저는 보고에 따로 적는다.
- 리뷰 품질이 낮게 느껴지면 Codex 설정 파일의 reasoning effort 기본값을 확인한다.
- 스킬 파일이 저장소 안 무시 경로에 있으면 다른 PC·다른 저장소 세션에는 없다.

## 참고
- OpenAI Codex CLI 문서: `codex exec`, sandbox 모드
- Claude Code 문서: Skills, Hooks
