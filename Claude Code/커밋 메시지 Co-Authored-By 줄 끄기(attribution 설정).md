# 커밋 메시지 Co-Authored-By 줄 끄기(attribution 설정)

> 분류: Claude Code · 최근 갱신: 2026-11-03

## 한 줄 요약
Claude Code가 커밋 메시지·PR 본문 끝에 붙이는 `Co-Authored-By: Claude ...` 줄은 `settings.json`의 `attribution` 항목으로 끈다. 이미 만든 push 전 커밋은 메시지만 다시 써서 뺄 수 있다.

## 내용
- `attribution.commit`: 커밋 메시지 끝에 붙는 문구. 빈 문자열이면 붙이지 않는다.
- `attribution.pr`: PR 본문 끝 문구. 빈 문자열이면 붙이지 않는다.
- `attribution.sessionUrl`: 세션 링크를 붙일지 여부.
- 사용자 전역 설정(`~/.claude/settings.json`)에 넣으면 모든 프로젝트·세션에 적용되고, 프로젝트 `.claude/settings.json`에 넣으면 그 저장소에만 적용된다.
- 설정은 앞으로 만드는 커밋에만 영향을 준다. 이미 만든 커밋은 그대로다.

### 이미 만든 커밋에서 빼기
- push 전이면 메시지만 다시 써도 된다. 코드·작성자·작성 시각은 유지되고 해시만 바뀐다.
- 마지막 커밋 하나면 `git commit --amend`, 여러 개면 `git rebase`로 메시지를 고친다. 바꾸기 전 해시와 새 해시 사이에 `git diff`가 비어 있는지로 코드가 그대로인지 확인한다.
- 이미 push한 커밋은 다시 쓰면 force-push가 필요하므로, 공유 브랜치에서는 그대로 두는 편이 낫다.

## 핵심 코드
### 1) settings.json
```json
{
  "attribution": {
    "commit": "",
    "pr": "",
    "sessionUrl": false
  }
}
```

### 2) push 전 커밋 메시지에서 줄 빼기
```bash
# 마지막 커밋
git commit --amend -m "$(git log -1 --format=%B | grep -v '^Co-Authored-By:')"

# 여러 커밋 (origin/main 이후 전부, 비대화형)
git -c core.editor=true rebase origin/main \
  --exec 'git commit --amend -m "$(git log -1 --format=%B | grep -v "^Co-Authored-By:")"'

# 코드가 그대로인지 확인 (출력이 없어야 함)
git diff <이전 해시> <새 해시>
```

## 주의할 점
- `--exec`로 amend하면 작성자·작성 시각은 유지되지만 커미터 시각은 바뀐다.
- 메시지에서 줄을 지울 때 빈 줄이 끝에 남을 수 있다. `git log -1`로 결과를 확인한다.

## 참고
- Claude Code 문서: Settings (attribution)
