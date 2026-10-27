# Agent Skills SKILL.md 위치(Claude Code와 Codex)

> 분류: Claude Code · 최근 갱신: 2026-10-27

## 한 줄 요약
스킬 파일명은 `SKILL.md`가 표준이지만 도구마다 찾는 폴더가 다르다. Claude Code는 `.claude/skills/`, OpenAI Codex는 `.agents/skills/`를 읽으므로, 한쪽에만 두면 다른 도구는 스킬로 자동 등록하지 않는다.

## 내용
- Agent Skills는 Anthropic이 공개 표준으로 내놓은 형식이다. 스킬 하나가 폴더 하나이고, 그 안에 `SKILL.md`(대문자, 이 이름 그대로)를 둔다. 파일 맨 위 YAML 머리말에 `name`, `description`이 있어야 한다. 여러 에이전트 도구가 이 형식을 지원한다고 알려져 있다(지원 목록은 2차 자료 기준).
- 읽는 위치

| 도구 | 저장소 단위 | 사용자 단위 |
|---|---|---|
| Claude Code | `.claude/skills/<이름>/SKILL.md` | `~/.claude/skills/` |
| Codex | `.agents/skills/<이름>/SKILL.md` (실행 폴더부터 저장소 루트까지 올라가며 찾음) | `$HOME/.agents/skills` |

- Codex 공식 문서에는 `.claude/skills`를 읽는다는 내용이 없다. 다만 Codex는 `AGENTS.md`를 기본으로 읽으므로, `AGENTS.md`에 스킬 파일 경로를 적어 두면 지시를 따라 파일을 직접 열 수는 있다. 이것은 "자동 등록"이 아니라 "문서를 따라가서 읽기"라 상황에 맞춘 자동 호출은 없다.
- 스킬은 해당 작업이 시작될 때(설명과 맞을 때) 읽힌다. 항상 지켜야 하는 한 줄 규칙은 모든 도구가 매번 읽는 `AGENTS.md`/`CLAUDE.md`에 두고, 스킬에는 절차·체크리스트를 두는 식으로 나누는 게 좋다.
- 나누는 기준은 "매 세션 읽어야 하는 규칙인가, 특정 작업 때만 필요한 절차인가"이다. 문서 양식, 배치 작업 골격 코드, 검증 명령 모음처럼 가끔 쓰는 것은 스킬로 뺀다. 다만 스킬을 안 부르고 작업하는 실수를 막으려면, 꼭 지킬 금지 규칙 한두 줄과 "만들 때는 ○○ 스킬을 따른다"는 참조는 지침 파일에 남긴다. 이렇게 정리하면 지침 파일이 절반 가까이 줄어든다.
- Claude Code는 `CLAUDE.md`만 자동으로 읽는다. `AGENTS.md`를 쓰려면 `CLAUDE.md`에 `@AGENTS.md` import를 넣는다. 이미 열린 세션에는 반영되지 않고 새 세션부터 적용된다.

## 핵심 코드
### 1) 폴더 구조
```text
<저장소>/
  AGENTS.md                 # 공통 핵심 규칙 (Codex 기본, Claude는 CLAUDE.md에서 @AGENTS.md로 포함)
  CLAUDE.md
  .claude/skills/verify/SKILL.md
  .agents/skills -> ../.claude/skills   # 링크로 두 도구가 같은 파일을 보게
```

### 2) CLAUDE.md에서 AGENTS.md 포함
```markdown
# <프로젝트>

@AGENTS.md
```
(새 세션부터 적용된다.)

### 3) SKILL.md 작성
```markdown
---
name: verify
description: 코드 변경 후 빌드·테스트·정적 확인 절차. "검증해줘" 요청 시 사용.
---

# 검증 절차
1. 빌드: `<빌드 명령>`
2. 변경한 기능의 테스트: `<테스트 명령>`
3. 결과 보고: 통과/실패, 확인하지 못한 항목
```
`description`에는 "무엇을 하는지"와 "언제 쓰는지(요청 문구)"를 함께 적는다. 모델은 이 설명만 보고 스킬을 고른다.

### 4) 두 도구가 같은 폴더를 보게 링크
```bat
rem Windows 폴더 링크 (저장소 루트에서, 관리자 권한 또는 개발자 모드 필요)
mkdir .agents
mklink /D .agents\skills ..\.claude\skills
rem 권한 없이 쓸 수 있는 디렉터리 정션 (같은 드라이브)
mklink /J .agents\skills .claude\skills
```
```bash
# macOS / Linux
mkdir -p .agents && ln -s ../.claude/skills .agents/skills
```

### 5) 공유 상태 확인
```bash
ls .claude/skills/*/SKILL.md .agents/skills/*/SKILL.md   # 두 경로에서 같은 파일이 보이는지
git check-ignore -v .claude/skills/verify/SKILL.md      # ignore 규칙에 걸리면 그 규칙이 출력된다
git ls-files AGENTS.md CLAUDE.md .claude/skills          # 실제로 추적 중인 파일
```

### 6) 에이전트 파일만 골라 무시하는 .gitignore
```gitignore
# *.md 전체가 아니라 대상만 적는다
AGENTS.md
CLAUDE.md
.claude/**/*.md
```

## 주의할 점
- 링크 대신 복사하면 두 곳을 같이 고쳐야 해서 내용이 어긋나기 쉽다.
- `.gitignore`에 스킬 폴더가 걸려 있으면 그 PC에서만 적용되고 팀원에게는 공유되지 않는다.
- 에이전트 파일을 저장소 밖에 두려고 `*.md` 전체를 무시하면, 라이브러리에 딸린 README·LICENSE까지 걸린다. `AGENTS.md`, `CLAUDE.md`, `.claude/**/*.md`처럼 대상만 적는다. 이미 추적 중인 파일은 ignore 규칙과 관계없이 계속 추적된다.

## 참고
- Claude Code 문서: Agent Skills
- OpenAI Codex 문서: Skills, AGENTS.md
