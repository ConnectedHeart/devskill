# git 한글 경로 깨짐과 스테이징 후 삭제된 파일

> 분류: Git · 최근 갱신: 2026-10-27

## 한 줄 요약
`git status`에서 한글 경로가 `"\354\273\244..."`처럼 보이는 것은 `core.quotepath=true`(기본값) 때문이며 `git config --global core.quotepath false`로 고친다. 스테이징한 뒤 디스크에서 지운 파일은 "커밋 예정"과 "삭제됨"에 동시에 나오고, 스테이징 영역에서 되살릴 수 있다.

## 내용
### 한글 경로 표시
- git은 기본으로 경로의 비ASCII 바이트를 8진수 이스케이프로 출력한다. 파일이 깨진 것이 아니라 표시 방식이다.
- `core.quotepath false`로 바꾸면 UTF-8 그대로 출력한다. 그래도 `???`나 깨진 글자면 터미널 인코딩 문제다(Git Bash: 창 제목 우클릭 → Options → Text → Locale `ko_KR`, Character set `UTF-8`).
- 커밋 메시지 인코딩은 별도 설정(`i18n.commitEncoding`, `i18n.logOutputEncoding`)이다.

### 스테이징 후 삭제된 파일
- `git add`로 새 파일을 스테이징한 뒤 디스크에서 지우면, 인덱스에는 내용이 있고 작업 폴더에는 없다. 그래서 `Changes to be committed: new file`과 `Changes not staged: deleted`에 함께 나온다.
- 내용은 인덱스에 남아 있어 `git restore <경로>`로 되살릴 수 있다. 지운 게 맞다면 `git rm --cached <경로>`로 인덱스에서도 빼며, 이때 내용은 사라진다.
- 커밋 전에는 스테이징된 목록에 설정 파일·로그 같은 올리면 안 되는 파일이 섞였는지 확인하고 `git restore --staged <경로>`로 뺀다.

## 핵심 코드
```bash
# 한글 경로 그대로 보기 (모든 저장소)
git config --global core.quotepath false
# 커밋 메시지 인코딩
git config --global i18n.commitEncoding utf-8
git config --global i18n.logOutputEncoding utf-8
```

```bash
# 스테이징 후 디스크에서 지운 파일
git restore "docs/분석 노트.md"          # 인덱스 내용으로 작업 폴더에 되살리기
git rm --cached "docs/분석 노트.md"      # 지운 게 맞으면 인덱스에서도 제거(내용 사라짐)

# 수정된 파일
git add "업무/절차서.docx"               # 올릴 거면 스테이징
git restore "업무/절차서.docx"           # 커밋된 버전으로 되돌리기(로컬 수정 사라짐)

# 실수로 스테이징한 파일 빼기
git restore --staged "logs/server.log"
```

## 주의할 점
- `--global`은 그 PC의 모든 저장소에 적용된다. 한 저장소만 바꾸려면 `--global` 없이 실행한다.
- `git restore <경로>`는 작업 폴더의 로컬 수정을 되돌릴 수 없게 덮어쓴다. 바이너리 문서는 먼저 사본을 둔다.
- 경로에 공백·한글이 있으면 따옴표로 감싼다.

## 참고
- Git 문서: git-config(core.quotepath, i18n.*), git-restore, git-rm(--cached)
