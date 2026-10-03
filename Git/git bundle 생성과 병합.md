# git bundle 생성과 병합

> 분류: Git · 최근 갱신: 2026-10-01

## 한 줄 요약
`git bundle`은 저장소(또는 일부 브랜치)를 파일 하나로 묶어 네트워크 없이 옮기는 방법이고, 받는 쪽에서는 그 파일을 원격 저장소처럼 clone/fetch 한다.

## 내용
### 저장소를 다른 PC로 옮기는 방법 비교
| 방법 | 옮겨지는 것 | 한계 |
|---|---|---|
| 작업 폴더만 압축(`.git` 제외) | 현재 브랜치의 파일 | 이력 조회, 브랜치 전환 불가 |
| `.git` 포함 폴더 복사 | 이력 + 커밋 안 한 수정 | 빌드 결과물까지 따라와 용량이 큼 |
| `git bundle --all` | 로컬 브랜치, 원격 추적 브랜치, 태그 | 커밋 안 한 수정은 안 들어감, stash는 가장 최근 1개만 |

### 번들 만들기
형식은 `git bundle create <파일> <담을 범위>`이다.
- 파일 이름은 임의이고 뒤에 적는 브랜치명과 무관하다. `.bundle`은 관례상 확장자다.
- 브랜치 하나를 담으면 그 브랜치 끝에서 거슬러 올라가 닿는 모든 커밋과 **그 브랜치 이름 하나**가 들어간다. 조상 커밋은 들어가도 다른 브랜치의 이름은 없다.
- 증분 번들(`기준..브랜치`)은 기준에 없는 커밋만 담는다. 받는 쪽에 기준 커밋이 없으면 `git bundle verify`가 "lacks these prerequisite commits"로 실패한다.
- 전체 번들은 `git clone`이 되지만 증분 번들은 fetch만 된다.

### 경로로 직접 fetch하면 FETCH_HEAD에만 남는다
- 등록된 원격(`git fetch origin`)은 설정된 규칙에 따라 `origin/브랜치`를 갱신한다.
- 경로나 URL을 직접 주면 그런 규칙이 없어서, 커밋만 저장소에 넣고 끝 커밋을 `.git/FETCH_HEAD`에 적어 둔다. 로컬 브랜치는 생기지 않는다.
- 그래서 fetch 뒤에 `git merge <브랜치명>`을 하면 번들 내용이 아니라 **예전부터 있던 같은 이름의 로컬 브랜치**가 병합되거나, 없으면 "not something we can merge" 오류가 난다. `git merge FETCH_HEAD`를 써야 한다.
- `FETCH_HEAD`는 다음 fetch/pull 때 덮어써지므로 fetch 직후에 병합한다. `git pull <경로> <브랜치>`가 내부적으로 하는 일이 `fetch` + `merge FETCH_HEAD`다.

### --no-ff
- fast-forward는 내 브랜치에 새 커밋이 없을 때 병합 커밋 없이 포인터만 옮기는 기본 동작이다. 이력이 한 줄이 되어 어디까지가 들어온 커밋인지 남지 않는다.
- `--no-ff`는 병합 커밋을 반드시 만든다. 합친 시점이 남고 `git revert -m 1 <병합 커밋>` 한 번으로 묶음 전체를 되돌릴 수 있다.
- 양쪽이 이미 갈라져 있으면 어차피 병합 커밋이 생기므로 옵션 유무와 결과가 같다.

## 예시
```bash
# 만들기
git bundle create ../repo.bundle --all          # 전체
git bundle create ../feature.bundle feature     # 브랜치 하나
git bundle create ../inc.bundle main..feature   # 증분

# 새로 받기
git clone repo.bundle myrepo

# 기존 저장소에 병합
git bundle verify ../repo.bundle
git bundle list-heads ../repo.bundle
git fetch ../repo.bundle feature
git log --oneline HEAD..FETCH_HEAD
git merge FETCH_HEAD

# 이름을 붙여 받기
git fetch ../repo.bundle feature:bundle-feature
git merge bundle-feature
```

## 주의할 점
- 번들은 브랜치명으로 만든다. `HEAD`로 만들면 번들 안에 브랜치 이름이 없어 `git fetch <번들> <브랜치>`를 쓸 수 없다.
- 로컬 브랜치 기준으로 담긴다. 원격 최신을 담으려면 먼저 pull 한다.
- 번들 파일은 저장소 밖에 만들어야 실수로 커밋되지 않는다.
- 이력을 공유하지 않는 저장소끼리는 merge가 거부된다. `--allow-unrelated-histories`는 사실상 전체 충돌이므로 `git format-patch` → `git am`이 낫다.
- 저장소에 커밋된 설정 파일의 비밀값도 번들에 그대로 들어간다. 반출 전에 확인한다.
- 번들은 이미 압축되어 있어 분할할 때는 무압축으로 나눠도 된다. 윈도우 기본 압축 풀기는 분할 zip을 지원하지 않는다.

## 참고
- Git 공식 문서: git-bundle, git-fetch(FETCH_HEAD), git-merge(--no-ff)
