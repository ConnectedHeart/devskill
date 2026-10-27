# format-patch와 am으로 커밋 옮기기와 적용 전 검증

> 분류: Git · 최근 갱신: 2026-10-27

## 한 줄 요약
다른 브랜치나 다른 저장소로 커밋 몇 개를 넘길 때는 `git format-patch`로 패치 파일을 만들고 받는 쪽에서 `git am`으로 적용한다. 적용 전에는 체크아웃 없이 임시 인덱스로 `git apply --cached --check`를 돌려 충돌 여부를 미리 본다.

## 내용
### git am이란
- "apply mailbox"의 줄임말이다. 원래 메일로 받은 패치를 저장소에 넣으려고 만든 명령이다.
- `format-patch`로 만든 파일에는 `From`(작성자), `Date`(작성 시각), `Subject`(제목), 본문, diff가 모두 들어 있다. `git am`은 이 정보로 커밋을 만들므로 커밋 메시지를 따로 쓸 필요가 없다. 커미터만 적용한 사람이 된다.

| 명령 | 코드 변경 | 커밋 생성 | 메시지·작성자 |
|---|---|---|---|
| `git am` | 적용 | 패치마다 자동 | 패치 파일 것 그대로 |
| `git apply` | 적용 | 안 함 | 직접 커밋해야 함 |
| `git cherry-pick` | 적용 | 자동 | 원 커밋 것 그대로. 원 커밋이 같은 저장소에 있어야 함 |

- 같은 저장소 안에서는 `cherry-pick`, 패치 파일로 주고받을 때는 `git am`을 쓴다. 받는 쪽 소스가 git 저장소가 아니면 `git apply`로 파일만 바꾼다.

### 어느 커밋이 안 넘어갔는지 찾기
- `git log --cherry-pick --right-only --oneline 릴리즈..작업브랜치 --author=이름`: 체리픽으로 이미 들어간 커밋(패치 내용이 같은 것)은 빼고, 작업 브랜치에만 있는 커밋을 보여준다.
- 해시만 다르고 내용이 같은 커밋은 `--cherry-pick`이 걸러 준다. 공백만 다른 경우는 못 거르므로 제목이 같은 커밋이 있으면 `git diff`로 다시 비교한다.
- 로컬 기준 비교라는 점을 기억한다. 원격에 더 새 커밋이 있으면 먼저 fetch 한다.

### 패치 만들기
- `git format-patch -o <폴더> <해시>^..<해시>` 또는 `git format-patch -o <폴더> -1 <해시>`를 커밋마다 실행한다. 여러 커밋을 한 번에 만들면 `0001-`, `0002-` 순번이 붙고 그 순서로 적용해야 한다.
- 파일명은 제목의 ASCII 글자만 남기므로 한글 제목은 비어 보인다. 읽기 쉽게 이름을 바꿔도 패치 내용(헤더의 제목·작성자·날짜)은 그대로다.
- `git am`은 원래 작성자·날짜·메시지를 보존한다. `git apply`는 변경만 적용하고 커밋은 만들지 않는다.
- **제목 앞의 `[태그]`가 사라지는 문제**: `git am`은 기본으로 제목의 `[...]` 접두어를 지운다(`[PATCH]`를 지우는 동작). 커밋 제목이 `[모듈] 기능 - 요약`처럼 대괄호로 시작하면 그 태그까지 사라진다. `-k`(`--keep-subject`)는 `[PATCH]`까지 남기고, `--keep-non-patch`는 `[PATCH]`만 지우고 나머지 대괄호는 남긴다.
- **파일명을 커밋 제목으로 바꿀 때**: 번호를 앞에 두어야(`0001 제목.patch`) `*.patch` 글롭 순서가 적용 순서와 같다. Windows에서 쓸 수 없는 `\ / : * ? " < > |`는 전각 문자(`／ ： ？ ＞`)로 바꾸면 읽기 쉽고, 너무 긴 제목은 잘라 낸 뒤 전체 제목은 별도 목록 파일에 둔다.
- **선행 커밋 때문에 하나만 막힐 때**: 뒤 패치가 다른 사람 커밋이 만든 코드를 고친 것이면, 그 커밋을 원래 체리픽된 순서 자리에 선행 패치로 넣어야 한다. 막힌 패치 바로 앞이 아니라 원래 순서대로 넣어야 그 뒤 패치도 그대로 들어간다. 선행 커밋을 넣을 수 없으면 그 패치를 빼고 나머지만 다시 적용해 본다.

### 적용 전에 충돌 여부만 확인하기
- 릴리즈 브랜치를 체크아웃하지 않고 확인하려면 임시 인덱스를 쓴다. `GIT_INDEX_FILE`을 임시 파일로 지정하고 `git read-tree 릴리즈브랜치`로 그 브랜치의 트리를 읽어 들인 뒤 `git apply --cached --check 패치`를 돌린다. 작업 폴더와 실제 인덱스는 건드리지 않는다.
- `git worktree add`로 임시 작업 폴더를 만드는 방법도 있지만 Windows에서 긴 경로나 잠긴 파일로 실패할 수 있다. 그때 임시 인덱스 방법이 대안이다.
- `--check`가 통과해도 컴파일이 된다는 뜻은 아니다. 선행 커밋이 빠져 있으면 적용은 되고 빌드가 깨질 수 있다.
- 패치 여러 개를 순서대로 넣을 계획이면 `--check`만 따로 돌리지 말고, 임시 인덱스에 `git apply --cached`로 하나씩 실제 적용하며 다음 패치를 확인한다. 뒤 패치가 앞 패치의 변경을 전제로 하는 경우를 함께 볼 수 있다. 임시 인덱스라 작업 폴더는 그대로다.
- `space before tab in indent` 같은 공백 경고는 오류가 아니라 경고다. `git am`/`git apply`는 경고만 내고 적용한다(`--whitespace=fix`로 고치며 적용할 수도 있다).
- Git for Windows에서 `/c/...` 형태 경로나 특수문자가 많은 파일명을 못 읽을 때는 패치를 단순한 이름으로 복사해 넘긴다. 파일명에 쓸 수 없는 문자(`/`, `:` 등)가 제목에 있으면 이름을 바꿀 때 공백 등으로 바꾼다.

## 핵심 코드
### 1) 넘길 커밋 찾기
```bash
git fetch origin
# 작업 브랜치에만 있는 내 커밋 (같은 내용으로 이미 체리픽된 것은 제외)
git log --cherry-pick --right-only --oneline release...feature --author="홍길동"
git log --cherry-pick --right-only --no-merges --reverse --format='%h %ad %s' --date=short origin/release...origin/feature
```
(본문의 `릴리즈..작업브랜치`는 `--cherry-pick`이 비교하려면 점 세 개(`...`) 대칭 범위로 써야 한다.)

### 2) 패치 만들기
```bash
# 커밋마다 하나씩 (오래된 것부터)
git format-patch -o ~/Desktop/patches -1 <해시1>
git format-patch -o ~/Desktop/patches -1 <해시2>

# 연속된 범위를 한 번에: 0001-, 0002- 순번이 붙는다
git format-patch -o ~/Desktop/patches <첫해시>^..<마지막해시>

# 번호를 붙여 여러 개를 따로 만들 때 순번 지정
git format-patch -o ~/Desktop/patches --start-number 3 -1 <해시3>
```

### 3) 체크아웃 없이 임시 인덱스로 미리 확인
```bash
export GIT_INDEX_FILE=/tmp/idx          # 이 셸에서만 임시 인덱스 사용
git read-tree release                   # 받는 브랜치의 트리를 임시 인덱스에 읽기
for p in ~/Desktop/patches/*.patch; do
  if git apply --cached --check "$p"; then
    git apply --cached "$p"             # 다음 패치가 앞 패치 결과를 전제로 하므로 실제로 쌓는다
    echo "OK   $p"
  else
    echo "FAIL $p"; break
  fi
done
unset GIT_INDEX_FILE && rm -f /tmp/idx  # 실제 인덱스와 작업 폴더는 그대로
```

### 4) 받는 쪽에서 적용
```bash
git status --short                      # 커밋하지 않은 변경이 없어야 한다
git checkout release
git am --keep-non-patch ~/Desktop/patches/*.patch   # 제목의 [모듈] 태그 유지

# 0001만 빼고 적용 (Git Bash 중괄호 펼침: 0002 ~ 0010)
git am "C:/<경로>/patches/"00{02..10}*.patch

# 충돌로 멈췄을 때
git am --show-current-patch=diff        # 막힌 패치 내용
git am --abort                          # 이번 am 전체를 실행 전으로
git am --3way --keep-non-patch ~/Desktop/patches/*.patch   # 3-way로 충돌 표시를 만들어 적용
git add <해결한 파일> && git am --continue
git am --skip                           # 그 패치만 건너뜀
```

### 5) 임시 worktree에서 순서대로 실제 적용해 보기
```bash
git worktree add --detach C:/wt-try <기준커밋>
git -C C:/wt-try am --keep-non-patch ~/Desktop/patches/*.patch
git -C C:/wt-try log --oneline -5
git worktree remove --force C:/wt-try
```

### 6) PowerShell에서 적용
```powershell
# 전체 적용 / 특정 패치 제외
git am (Get-ChildItem "<경로>\patches\*.patch").FullName
git am (Get-ChildItem "<경로>\patches\*.patch" | Where-Object Name -notlike '0001*' | Sort-Object Name).FullName

# 체크아웃 없이 확인 (PowerShell)
$env:GIT_INDEX_FILE = "$env:TEMP\idx"
git read-tree release
git apply --cached --check (Get-ChildItem "<경로>\patches\*.patch" | Sort-Object Name).FullName
Remove-Item Env:GIT_INDEX_FILE; Remove-Item "$env:TEMP\idx"
```

## 주의할 점
- 패치 파일에는 변경된 줄이 그대로 들어간다. 설정 파일의 비밀값이 커밋에 포함돼 있었다면 패치에도 들어간다. 반출 전에 내용을 본다.
- `git am`이 충돌로 멈추면 그 앞 패치들은 이미 커밋된 상태다. `git am --abort`(이번 am 전체를 실행 전으로), `git am --3way <파일>`(3-way 병합으로 충돌 표시 → 해결 후 `git add` → `git am --continue`), `git am --skip`(그 패치만 건너뜀) 중 하나로 정리한다. 중간 상태로 두면 다음 작업이 꼬인다.
- 실행 전에 작업 폴더에 커밋하지 않은 변경이 없어야 한다.
- Git Bash에서 `git am`에 `/c/...` 형태 경로를 주면 파일을 못 여는 경우가 있다. `C:/...` 형식으로 준다.
- 커밋 하나를 통째로 옮기는 용도다. 일부만 필요하면 `git apply` 후 직접 커밋한다.

## 참고
- Git 문서: git-format-patch, git-am, git-apply(--cached --check), git-log(--cherry-pick), git-read-tree, GIT_INDEX_FILE
