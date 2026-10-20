# format-patch와 am으로 커밋 옮기기와 적용 전 검증

> 분류: Git · 최근 갱신: 2026-10-19

## 한 줄 요약
다른 브랜치나 다른 저장소로 커밋 몇 개를 넘길 때는 `git format-patch`로 패치 파일을 만들고 받는 쪽에서 `git am`으로 적용한다. 적용 전에는 체크아웃 없이 임시 인덱스로 `git apply --cached --check`를 돌려 충돌 여부를 미리 본다.

## 내용
### 어느 커밋이 안 넘어갔는지 찾기
- `git log --cherry-pick --right-only --oneline 릴리즈..작업브랜치 --author=이름`: 체리픽으로 이미 들어간 커밋(패치 내용이 같은 것)은 빼고, 작업 브랜치에만 있는 커밋을 보여준다.
- 해시만 다르고 내용이 같은 커밋은 `--cherry-pick`이 걸러 준다. 공백만 다른 경우는 못 거르므로 제목이 같은 커밋이 있으면 `git diff`로 다시 비교한다.
- 로컬 기준 비교라는 점을 기억한다. 원격에 더 새 커밋이 있으면 먼저 fetch 한다.

### 패치 만들기
- `git format-patch -o <폴더> <해시>^..<해시>` 또는 `git format-patch -o <폴더> -1 <해시>`를 커밋마다 실행한다. 여러 커밋을 한 번에 만들면 `0001-`, `0002-` 순번이 붙고 그 순서로 적용해야 한다.
- 파일명은 제목의 ASCII 글자만 남기므로 한글 제목은 비어 보인다. 읽기 쉽게 이름을 바꿔도 패치 내용(헤더의 제목·작성자·날짜)은 그대로다.
- `git am`은 원래 작성자·날짜·메시지를 보존한다. `git apply`는 변경만 적용하고 커밋은 만들지 않는다.

### 적용 전에 충돌 여부만 확인하기
- 릴리즈 브랜치를 체크아웃하지 않고 확인하려면 임시 인덱스를 쓴다. `GIT_INDEX_FILE`을 임시 파일로 지정하고 `git read-tree 릴리즈브랜치`로 그 브랜치의 트리를 읽어 들인 뒤 `git apply --cached --check 패치`를 돌린다. 작업 폴더와 실제 인덱스는 건드리지 않는다.
- `git worktree add`로 임시 작업 폴더를 만드는 방법도 있지만 Windows에서 긴 경로나 잠긴 파일로 실패할 수 있다. 그때 임시 인덱스 방법이 대안이다.
- `--check`가 통과해도 컴파일이 된다는 뜻은 아니다. 선행 커밋이 빠져 있으면 적용은 되고 빌드가 깨질 수 있다.
- 패치 여러 개를 순서대로 넣을 계획이면 `--check`만 따로 돌리지 말고, 임시 인덱스에 `git apply --cached`로 하나씩 실제 적용하며 다음 패치를 확인한다. 뒤 패치가 앞 패치의 변경을 전제로 하는 경우를 함께 볼 수 있다. 임시 인덱스라 작업 폴더는 그대로다.
- `space before tab in indent` 같은 공백 경고는 오류가 아니라 경고다. `git am`/`git apply`는 경고만 내고 적용한다(`--whitespace=fix`로 고치며 적용할 수도 있다).
- Git for Windows에서 `/c/...` 형태 경로나 특수문자가 많은 파일명을 못 읽을 때는 패치를 단순한 이름으로 복사해 넘긴다. 파일명에 쓸 수 없는 문자(`/`, `:` 등)가 제목에 있으면 이름을 바꿀 때 공백 등으로 바꾼다.

## 예시
```bash
# 작업 브랜치에만 있는 내 커밋
git log --cherry-pick --right-only --oneline release..feature --author="홍길동"

# 패치 생성 (오래된 것부터)
git format-patch -o ~/Desktop/patches -1 <해시1>
git format-patch -o ~/Desktop/patches -1 <해시2>

# 체크아웃 없이 적용 가능 여부 확인
GIT_INDEX_FILE=/tmp/idx git read-tree release
GIT_INDEX_FILE=/tmp/idx git apply --cached --check ~/Desktop/patches/*.patch

# 받는 쪽에서 적용
git checkout release
git am ~/Desktop/patches/*.patch
```

## 주의할 점
- 패치 파일에는 변경된 줄이 그대로 들어간다. 설정 파일의 비밀값이 커밋에 포함돼 있었다면 패치에도 들어간다. 반출 전에 내용을 본다.
- `git am`이 충돌로 멈추면 `git am --abort`로 되돌리거나 충돌을 고친 뒤 `git am --continue`한다. 중간 상태로 두면 다음 작업이 꼬인다.
- 커밋 하나를 통째로 옮기는 용도다. 일부만 필요하면 `git apply` 후 직접 커밋한다.

## 참고
- Git 문서: git-format-patch, git-am, git-apply(--cached --check), git-log(--cherry-pick), git-read-tree, GIT_INDEX_FILE
