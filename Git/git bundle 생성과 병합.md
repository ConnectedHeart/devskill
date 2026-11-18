# git bundle 생성과 병합

> 분류: Git · 최근 갱신: 2026-11-18

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

### 갈라진 브랜치만 작게 담기와 이력 없는 스냅샷
- 다른 저장소에서 갈라져 나온 브랜치(사이트 전용 브랜치 등)를 넘길 때, 받는 쪽에 원본 저장소가 있으면 **두 쪽이 마지막으로 같이 가진 커밋**(`git merge-base`) 이후만 담는다. 전체 1GB짜리 이력이 수십 MB로 준다. 받는 쪽 원본이 그 커밋보다 오래됐으면 verify가 선행 커밋 없음으로 실패하므로 먼저 fetch하게 한다.
- 받는 쪽에 아무것도 없고 이력도 필요 없으면 `commit-tree`로 현재 트리만 담은 부모 없는 커밋을 만들어 번들로 담는다. 작업 폴더는 건드리지 않는다. 대신 받는 쪽에서 원본과 병합·체리픽은 할 수 없다.
- 번들 파일로 clone하면 `origin`이 그 파일 경로를 가리킨다. 파일을 옮기거나 지우면 fetch가 실패하므로 실제 원격이 생기면 `remote set-url`로 바꾼다.
- 지금 체크아웃된 브랜치 이름으로는 `fetch <번들> 브랜치:브랜치`가 거부된다. 다른 이름으로 받거나 `git pull <번들> <브랜치>`를 쓴다.
- 받는 쪽 브랜치 이름과 겹치지 않게 `fetch <번들> 브랜치:refs/remotes/<이름>/브랜치`처럼 원격 추적 이름으로 받아 병합하면, 상대가 보낸 것과 내 브랜치가 구분된다.

### 받은 번들을 원격 서버 브랜치에 올리기
- "번들로 원격 브랜치를 덮어쓴다"는 요청이라도 먼저 원격을 fetch해 번들 끝 이후 원격에 새 커밋이 있는지(`git log <번들끝>..<원격>/<브랜치>`) 본다. 없으면 일반 push로 이어 붙이고, 있으면 그 위로 rebase한 뒤 push한다. `--force`는 그 사이 다른 사람이 올린 커밋을 지운다.
- 정말 덮어써야 하면 먼저 원격 끝을 백업 브랜치로 push하고, `--force-with-lease=<브랜치>:<기대 SHA>`를 쓴다. 원격이 기대한 SHA가 아니면 거부되므로 `--force`보다 안전하다. 서버에 `receive.denyNonFastForwards`가 켜져 있으면 강제 push 자체가 막힌다.
- 번들에 빠진 최근 커밋은 번들 클론에 `git am`으로 더하고, 그 몇 건만 담은 증분 번들을 따로 만들면 다른 PC에서도 반영할 수 있다.

### 갈라진 로컬 브랜치를 원격에 맞추기
- 같은 내용을 서로 다른 경로(패치 적용, 체리픽)로 받아 해시만 다르면, 로컬과 원격이 "앞서고 뒤처진" 상태가 된다. 이때 pull은 같은 내용을 병합 커밋으로 섞고, push는 거절되거나 강제로 하면 원격 커밋이 지워진다.
- `git cherry <원격> <로컬>`로 로컬에만 있는 커밋이 모두 원격에 같은 내용(`-` 표시)으로 있는지 확인한 뒤 `git reset --keep <원격>/<브랜치>`로 맞춘다. `--keep`은 커밋하지 않은 수정을 남기고, 그 파일이 두 커밋 사이에서 바뀌었으면 아무것도 바꾸지 않고 멈춘다. 이전 끝은 reflog로 되돌릴 수 있다.
- 원격 주소를 바꾸면(`remote set-url`) fetch prune이 이전 원격에만 있던 원격 추적 브랜치를 로컬 목록에서 지운다(서버 쪽 브랜치는 그대로).
- 원격에 대소문자만 다른 브랜치가 있으면 Windows(대소문자 무시 파일 시스템)에서 fetch 때마다 오류가 나고 그 브랜치만 못 받는다. 다른 브랜치는 정상으로 받아진다.

### --no-ff
- fast-forward는 내 브랜치에 새 커밋이 없을 때 병합 커밋 없이 포인터만 옮기는 기본 동작이다. 이력이 한 줄이 되어 어디까지가 들어온 커밋인지 남지 않는다.
- `--no-ff`는 병합 커밋을 반드시 만든다. 합친 시점이 남고 `git revert -m 1 <병합 커밋>` 한 번으로 묶음 전체를 되돌릴 수 있다.
- 양쪽이 이미 갈라져 있으면 어차피 병합 커밋이 생기므로 옵션 유무와 결과가 같다.

## 핵심 코드
### 1) 번들 만들기 (보내는 쪽)
```bash
git switch feature && git pull          # 원격 최신을 담으려면 먼저 받는다
git bundle create ../repo.bundle --all          # 전체
git bundle create ../feature.bundle feature     # 브랜치 하나
git bundle create ../inc.bundle main..feature   # 증분 (받는 쪽에 main 끝 커밋이 있어야 함)
git bundle verify ../feature.bundle             # 만든 직후 확인
git rev-parse feature                           # 보고용 끝 커밋 해시
```

### 2) 새로 받기 (받는 쪽, 저장소 없음)
```bash
git clone repo.bundle myrepo
git clone release.bundle -b release myrepo   # 브랜치 하나만 담은 번들은 -b 로 그 브랜치를 지정
git bundle verify release.bundle             # 완전한 이력인지 확인

# 나중에 원래 원격으로 바꿔 연결
git -C myrepo remote set-url origin <원격 URL>
```

### 3) 기존 저장소에 병합
```bash
git bundle verify ../repo.bundle
git bundle list-heads ../repo.bundle
git fetch ../repo.bundle feature
git log --oneline HEAD..FETCH_HEAD       # 들어올 커밋 미리 보기
git merge --no-ff FETCH_HEAD             # fetch 직후 바로 병합 (FETCH_HEAD는 다음 fetch 때 덮어써짐)
```

### 4) 이름을 붙여 받기
```bash
git fetch ../repo.bundle feature:bundle-feature
git merge bundle-feature

# 번들을 원격처럼 등록해 두고 여러 번 받기
git remote add usb /<경로>/repo.bundle
git fetch usb                            # usb/<브랜치> 로 받아짐
```

### 5) 갈라진 브랜치만 담기 / 이력 없는 스냅샷
```bash
# 원본(main)과 마지막으로 공유한 커밋 이후만 (받는 쪽에 원본 저장소가 있을 때)
base=$(git merge-base main site-branch)
git bundle create ../site.bundle "$base"..site-branch
git -C <받는 쪽 저장소> bundle verify ../site.bundle   # is okay 확인

# 받는 쪽에서 원격 추적 이름으로 받아 병합
git fetch ../site.bundle site-branch:refs/remotes/vendor/site-branch
git merge vendor/site-branch
```
```powershell
# 이력 없이 현재 소스만 (작업 폴더는 그대로)
$c = git commit-tree 'site-branch^{tree}' -m 'snapshot'
git branch snapshot $c
git bundle create ..\snapshot.bundle snapshot
git branch -D snapshot
# 받는 쪽: git clone -b snapshot snapshot.bundle site-src
```

### 6) 번들 내용을 원격 서버 브랜치에 올리기
```bash
git fetch ../repo.bundle main:from-bundle                 # 번들을 임시 브랜치로
git remote add company <원격 URL>
git fetch company main
git log --oneline from-bundle..company/main               # 번들 이후 원격에 생긴 커밋
git push company from-bundle:main                         # 위가 비어 있으면 일반 push
# 원격에 새 커밋이 있으면: git rebase company/main from-bundle 후 push
# 꼭 덮어써야 할 때만 (백업 먼저)
git push company <옛SHA>:refs/heads/main_backup
git push --force-with-lease=main:<옛SHA> company from-bundle:main
```

### 7) 해시만 다른 로컬 브랜치를 원격에 맞추기
```bash
git fetch origin
git cherry origin/main main          # '+'가 하나도 없으면 로컬 전용 커밋 내용이 모두 원격에 있음
git reset --keep origin/main         # 미커밋 수정 유지, 충돌하면 멈춤
git reflog -3                        # 되돌릴 이전 끝 확인
```

### 8) 큰 번들 나누기와 합치기
```bash
split -b 1000m -d ../repo.bundle ../repo.bundle.part   # repo.bundle.part00, part01 ...
cat repo.bundle.part* > repo.bundle                     # 받는 쪽에서 합치기
git bundle verify repo.bundle
```

## 주의할 점
- 번들은 브랜치명으로 만든다. `HEAD`로 만들면 번들 안에 브랜치 이름이 없어 `git fetch <번들> <브랜치>`를 쓸 수 없다.
- 로컬 브랜치 기준으로 담긴다. 원격 최신을 담으려면 먼저 pull 한다. 번들 결과를 보고할 때는 끝 커밋 해시와 "fetch 안 했음"을 같이 적는다.
- 오래된 대형 저장소는 브랜치 하나를 담아도 전체 이력이 들어가 수백 MB가 된다. 받는 쪽이 이미 같은 이력을 갖고 있으면 증분 번들(`기준..브랜치`)이 훨씬 작다.
- 번들 파일은 저장소 밖에 만들어야 실수로 커밋되지 않는다.
- 이력을 공유하지 않는 저장소끼리는 merge가 거부된다. `--allow-unrelated-histories`는 사실상 전체 충돌이므로 `git format-patch` → `git am`이 낫다.
- 저장소에 커밋된 설정 파일의 비밀값도 번들에 그대로 들어간다. 반출 전에 확인한다.
- 번들은 이미 압축되어 있어 분할할 때는 무압축으로 나눠도 된다. 윈도우 기본 압축 풀기는 분할 zip을 지원하지 않는다.

## 참고
- Git 공식 문서: git-bundle, git-fetch(FETCH_HEAD), git-merge(--no-ff)
