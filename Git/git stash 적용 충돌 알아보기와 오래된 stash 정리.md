# git stash 적용 충돌 알아보기와 오래된 stash 정리

> 분류: Git · 최근 갱신: 2026-11-02

## 한 줄 요약
병합 중도 아닌데 충돌 파일이 남아 있고 표시가 `Updated upstream`/`Stashed changes`이면 stash 적용(apply/pop)에서 난 충돌이다. 오래된 stash가 이미 커밋된 작업의 초안이면 HEAD 쪽으로 풀고, 다시 적용되지 않게 내용을 확인한 뒤 지운다.

## 내용
### 어디서 난 충돌인지 가리기
- `.git/MERGE_HEAD`, `CHERRY_PICK_HEAD`, `rebase-merge/`가 없으면 병합·체리픽·리베이스 중이 아니다.
- 충돌 표시가 `<<<<<<< Updated upstream` … `>>>>>>> Stashed changes`이면 stash 적용 충돌이다. `git stash pop`은 충돌이 나면 stash를 지우지 않으므로 목록에 그대로 남는다.
- stash를 만든 경위는 reflog로 본다. `git stash`는 `reset: moving to HEAD` 기록을 남기므로, 그 직후의 `checkout: moving from A to B`를 보면 "A 브랜치에서 작업하다 B로 옮기려고 치워 둔 것"임을 알 수 있다. stash 커밋의 부모가 만든 시점의 기준 커밋이다.

### 충돌 해결 판단
- stash 쪽에만 있는 줄이 나중 커밋에서 일부러 지운 것(기능 제거)인지, 지금 쓰는 곳이 없는지 확인한다. 초안이 이미 다듬어져 커밋됐다면 HEAD 쪽(Updated upstream)만 남기면 된다.
- 해결한 파일이 HEAD와 완전히 같으면 `git add`로 해결 표시만 하고 커밋할 변경은 생기지 않는다(필요하면 `git restore --staged`로 인덱스도 정리).

### 오래된 stash 정리
- stash가 추가한 줄을 하나씩 현재 브랜치와 대조해, 살려야 할 내용이 없으면 지운다. 남겨 두면 누군가 다시 pop/apply해서 같은 충돌이 난다.
- `git stash drop`은 되돌리기 어렵다(지운 직후라면 출력된 해시나 `git fsck --unreachable`로 찾을 수는 있음). 지우기 전에 필요하면 `git stash show -p > stash-backup.patch`로 보관한다.

## 핵심 코드
```bash
# 1) 무엇 중인지 확인
ls .git/MERGE_HEAD .git/CHERRY_PICK_HEAD .git/rebase-merge 2>/dev/null
git diff --name-only --diff-filter=U            # 충돌 파일
grep -n 'Stashed changes' <파일>                 # stash 적용 충돌 표시

# 2) stash 경위와 내용
git stash list --date=iso
git log -g --date=iso --format='%gd %gs' HEAD | head -20   # reset: moving to HEAD 직후 checkout 기록
git log -1 --format='%H %s' stash@{0}^           # 만든 시점의 기준 커밋
git stash show --stat stash@{0}
git diff HEAD stash@{0} -- <경로>                # 현재 브랜치와 차이

# 3) HEAD 쪽으로 해결
git checkout --ours -- <충돌 파일>                # stash 적용 충돌에서 ours = HEAD(Updated upstream)
git add <충돌 파일>
git diff --cached --stat                          # 비어 있으면 커밋할 변경 없음

# 4) 보관 후 삭제
git stash show -p stash@{0} > stash-backup.patch
git stash drop stash@{0}
```

## 주의할 점
- stash 적용 충돌에서 `--ours`는 현재 HEAD, `--theirs`는 stash 쪽이다. 리베이스와 의미가 반대로 느껴질 수 있으니 결과 파일을 다시 확인한다.
- 줄 끝(CRLF)·BOM이 섞인 파일은 해결 후 HEAD와 바이트 단위로 같은지 `git diff`로 확인한다.

## 참고
- Git 공식 문서: git-stash, git-reflog, git-checkout(--ours/--theirs)
