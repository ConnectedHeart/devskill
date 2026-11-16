# 패치가 이미 들어갔는지 내용으로 판정하기(apply --check -R)

> 분류: Git · 최근 갱신: 2026-11-12

## 한 줄 요약
다른 저장소로 넘긴 패치·커밋이 상대 트리에 실제로 들어갔는지는 커밋 제목이 아니라 diff 내용으로 본다. `git apply --check`(그대로 붙음 = 없음)와 `git apply --check -R`(거꾸로 붙음 = 있음)를 차례로 시험하고, 둘 다 실패하면 "일부 다름"으로 따로 본다.

## 내용
### 왜 제목으로는 부족한가
- 상대가 체리픽하며 제목을 바꾸거나, 여러 커밋을 한 커밋("○차 전달")으로 합쳐 받으면 제목 검색에 걸리지 않는다.
- 반대로 제목이 같아도 충돌을 다르게 풀었으면 내용이 다르다.
- 그래서 제목은 1차로 거르는 데만 쓰고, 걸리지 않은 것은 내용으로 판정한다.

### 판정 순서
| 시험 | 통과하면 |
|---|---|
| `git apply --check -R <패치>` | 패치 내용이 트리에 그대로 있음 → "들어 있음" |
| `git apply --check <패치>` | 아무것도 안 들어감 → "없음" |
| 둘 다 실패 | "일부 다름" → 파일별로 다시 본다 |

- "일부 다름"은 대개 둘 중 하나다. (1) 들어간 뒤 같은 줄을 다른 커밋이 다시 고침 (2) 상대 쪽이 같은 취지를 다른 모양으로 고침. 어느 파일에서 실패하는지 보고(`--include`로 파일 하나씩), 패치가 새로 만드는 파일이 이미 있으면 (1)일 가능성이 크다.
- 더 정량적으로 보려면 패치의 "추가한 줄(+)"이 대상 파일에 몇 % 남아 있는지 센다. 100%면 들어 있음, 0%면 없음, 중간이면 사람이 본다.
- 실제 작업 트리를 건드리지 않으려면 임시 인덱스(`GIT_INDEX_FILE`)나 별도 worktree에서 `--cached`로 시험한다.

### 여러 시점에서 보기
"언제부터 없었나"를 알려면 같은 판정을 받은 쪽의 여러 시점(이전 전달본, 최신본)에 돌린다. 모든 시점에서 없으면 나중에 되돌린 게 아니라 처음부터 받지 않은 것이다.

### 순서 있는 패치 묶음 시험 적용
빠진 패치 여러 개를 넣기 전에 임시 브랜치나 worktree에서 묶음 순서대로 `git am`해 본다. 중간에 멈추는 패치는 대개 상대가 같은 곳을 다르게 고친 것이라, 억지로 3-way로 합치면 같은 보정이 두 번 들어가는 식의 이상한 결과가 나올 수 있다. 그런 패치는 빼고 나머지를 전달한다.

## 핵심 코드
### 1) 한 패치 판정
```bash
p=0001-some-change.patch
if git apply --check -R "$p" 2>/dev/null; then echo "들어 있음  $p"
elif git apply --check "$p" 2>/dev/null; then echo "없음      $p"
else echo "일부 다름  $p"; fi
```

### 2) 작업 트리를 건드리지 않고 다른 커밋 기준으로 판정
```bash
# 대상 커밋(target)의 트리를 임시 인덱스에 읽어 그 위에서만 시험
export GIT_INDEX_FILE=$(mktemp)
git read-tree target
git apply --cached --check -R "$p" && echo "들어 있음"
git apply --cached --check    "$p" && echo "없음"
rm -f "$GIT_INDEX_FILE"; unset GIT_INDEX_FILE
```

### 3) 이전 브랜치에만 있는 커밋을 diff로 다시 판정
```bash
# old에만 있는 커밋(제목 기준으로 걸러진 것)을 패치로 뽑아 new 트리에서 판정
git format-patch -o /tmp/only-old new..old
for p in /tmp/only-old/*.patch; do
  git apply --check -R "$p" 2>/dev/null && echo "있음 $p" && continue
  git apply --check    "$p" 2>/dev/null && echo "없음 $p" && continue
  echo "다름 $p"
done
```

### 4) 묶음 순서대로 시험 적용 (별도 worktree)
```bash
git worktree add -b trial ../trial target
cd ../trial
for p in $(ls /path/bundles/*/web/*.patch | sort); do
  git am --keep-non-patch "$p" || { echo "멈춤: $p"; git am --abort; break; }
done
```

## 주의할 점
- `--check`는 공백 차이에도 실패한다. 줄 끝 공백·CRLF가 다른 저장소면 `--ignore-whitespace`를 함께 시험한다.
- 작업 폴더 기준으로 판정하면 미커밋 변경이 섞인다. 커밋 기준이 필요하면 임시 인덱스나 worktree에서 한다.
- 제목 일치로 넘긴 패치는 내용까지 대조하지 않은 것이다. 결과 보고에 그 한계를 적는다.
- 상대가 받은 뒤 의도적으로 되돌린(Revert) 커밋도 "없음"으로 나온다. 되돌림 커밋이 있는지 같이 본다.

## 참고
- git-apply 문서 (`--check`, `-R`, `--cached`, `--include`)
- git-am 문서, git-worktree 문서
- 관련 노트: [format-patch와 am으로 커밋 옮기기와 적용 전 검증](format-patch와 am으로 커밋 옮기기와 적용 전 검증.md)
