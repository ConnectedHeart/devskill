# Redmine REST API로 일감 추출·검색·일괄 변경

> 분류: 기타 · 최근 갱신: 2026-11-07

## 한 줄 요약
Redmine은 화면 필터 형식을 REST API에서도 그대로 받으므로 일감 상세·첨부 추출, 조건 검색, 상태 일괄 변경을 스크립트로 할 수 있다. 다만 한 번에 100건, 워크플로 제약, 다른 Redmine으로 옮길 때 번호·작성자·시각은 그대로 못 옮긴다는 한계가 있다.

## 내용
### 준비
- 관리자가 REST API를 켜야 한다. 켜져 있으면 "내 계정" 화면에 **API 접근 키**가 보인다. 키는 `X-Redmine-API-Key` 헤더로 보낸다. 결과는 키 주인이 볼 수 있는 일감으로 제한된다.
- 일감 번호는 프로젝트별이 아니라 **사이트 전체에서 하나씩 증가**한다. 한 Redmine 안에서는 프로젝트가 달라도 번호가 겹치지 않지만, 다른 Redmine 사이트끼리는 겹칠 수 있다(합쳐 볼 때는 "주소 + 번호"로 구분).

### 추출
- `GET /issues/<id>.json?include=journals,attachments,relations`: 설명, 댓글·변경 이력(journals), 첨부 목록. 첨부 원본은 각 항목의 `content_url`로 받는다.
- 이력의 상태·담당자 변경은 ID로 오므로 `/issue_statuses.json`, `/trackers.json`, `/enumerations/issue_priorities.json`, `/projects/<id>/memberships.json`으로 이름 표를 만든다.
- 설명 안 이미지(`!그림.png!`)는 같은 이름의 첨부를 가리키므로, 첨부를 원래 이름으로 저장하면 오프라인 HTML에서도 제자리에 보여 줄 수 있다.

### 검색
- `GET /issues.json`에 화면과 같은 필터를 쓴다. 목록 화면 주소의 쿼리 문자열(`set_filter=1&f[]=...&op[...]=...&v[...][]=...`)을 그대로 붙여도 같은 결과가 온다(쪽 번호는 무시하고 직접 나눠 받는다).
- `limit` 최대 100, `offset`으로 나눠 받고 `total_count`로 끝을 안다.
- 같은 필드 안 여러 값은 OR(`status_id=2|6`), 필드끼리는 AND. 담당자 "없음"(`!*`)과 특정 사람을 한 필드에서 함께 고를 수는 없다.
- 저장된 검색은 `/queries.json`에서 id를 얻어 `query_id=<id>`로 쓴다.
- 목록형 사용자 정의 필드의 값 목록은 관리자 키가 아니면 403이다. 최근 일감 몇백 건을 훑어 나온 값을 모으는 식으로 대신한다.

### 변경
- `PUT /issues/<id>.json`에 `{"issue":{"status_id":..,"assigned_to_id":..,"notes":"..."}}`. 성공은 204.
- 워크플로가 허용하지 않는 상태 변경은 422로 거부되거나, 경우에 따라 **오류 없이 무시**된다. 반드시 다시 조회해 실제로 바뀌었는지 확인한다.
- 잘못된 값(예: 상태 ID 0)을 보내면 값은 무시돼도 "최종 수정 시각"은 바뀔 수 있다. 보내기 전에 ID가 목록에 있는지 검사한다.
- 담당자는 그 프로젝트 구성원이어야 한다. 이름으로 지정할 때는 구성원 목록에서 찾아 0건·동명이인이면 멈춘다.

### 오프라인 변경 요청(파일로 주고받기)
- 망이 분리된 곳에서는 "변경 요청 파일 → 현장에서 반영 → 결과 파일 회신"으로 한다. 요청에는 바꿀 값과 함께 **요청을 만들 때 본 값(seen)**을 넣는다. 반영 직전 다시 조회해 seen과 다르면 그 건은 자동 반영하지 않고 사람이 확인하게 한다. 이미 원하는 값이면 건너뛰어 같은 파일을 다시 돌려도 중복 댓글이 생기지 않게 한다.
- 형식은 JSON이 안전하다(한글·줄바꿈 메모·"안 바꿈"과 "비우기" 구분). 사람이 보기 불편하니 반영 툴에 미리 보기 표를 둔다. CSV는 인코딩·쉼표·빈 칸 의미 문제, 엑셀은 PowerShell 5.1 기본 기능으로 못 읽는 문제가 있다.

### 다른 Redmine으로 복제할 때
- 제목·설명·필드·첨부·댓글 내용은 옮길 수 있다. **번호, 작성자, 과거 작성 시각, 상태 변경 이력은 API로 그대로 못 옮긴다**(번호는 새로 붙고, 작성자는 키 주인, 이력은 실제 변경만 남음). "원본 번호" 사용자 정의 필드나 제목 접두어, 본문의 원 작성자·시각 표기로 대신한다.
- 반복 동기화는 원본 번호로 찾아 없으면 만들고 있으면 달라진 값만 고친다(같으면 아무것도 보내지 않아야 이력이 안 쌓인다). 댓글·첨부는 옮긴 원본 ID를 기록해 새것만 덧붙인다. 원본 수정 시각보다 오래된 추출본으로는 덮어쓰지 않는다. 원본에서 사라진 일감은 자동 삭제하지 말고 목록으로 알린다.
- 양방향은 같은 값을 두 곳에서 고치면 나중 반영이 이긴다. **항목마다 주인을 한쪽으로 정하고**(원본 쪽 항목은 사본에서 읽기 전용), 반대 방향은 변경 요청 파일로만 보낸다. 내가 보낸 댓글이 다음 추출 때 다시 들어오는 "메아리"는 결과 파일의 이력 ID로 걸러 낸다.
- 유형·상태·우선순위는 **이름으로** 맞춘다. 글자·띄어쓰기·대소문자까지 같아야 하고, 없으면 사본 설명에 글로 남기고 경고한다. 사본은 기본 상태(신규)로 만들어진 뒤 원본 상태로 바뀌므로, 대상 워크플로에서 복제 계정 역할이 **신규 → 모든 상태**로 갈 수 있어야 한다. 원본과 사본의 텍스트 서식(textile/markdown)이 다르면 본문 서식이 다르게 보인다.
- 설명을 덮어쓰면 이전 설명은 사본 쪽 이력에만(복제 시점 단위로) 남는다. 원본의 "설명 변경" 이력 내용 자체는 API로 받은 journal에 옛 값이 있어도 따로 옮기지 않으면 사라진다.

### 비공개 댓글 옮기기
- `notes`와 함께 `private_notes: true`를 보내도, 키 사용자에게 **비공개 댓글 작성 권한이 없으면 Redmine은 표시를 무시하고 일반 댓글로 올린다**(오류 없음). 내부 메모가 공개 댓글로 새는 원인이다.
- 그래서 올리기 전에 키 사용자의 역할 권한을 확인한다: `/users/current.json?include=memberships`로 역할 ID → `/roles/<id>.json`의 `permissions`에 `set_notes_private`(작성)·`view_private_notes`(보기)가 있는지. 확인이 안 되면(오래된 버전이라 API 없음 등) **내용을 올리지 않는 쪽**으로 처리한다. 관리자는 권한 검사 없이 통과한다.
- 올린 뒤 다시 조회해 그 journal의 `private_notes`가 true인지 확인한다. 보기 권한이 없으면 이미 옮긴 비공개 댓글이 조회에 안 보여 중복으로 다시 올릴 수 있다.

### 프로젝트 설정을 API로 맞출 수 있는 범위
| API로 가능 | 관리자 화면에서만 |
|---|---|
| 프로젝트 생성(이름·식별자·공개·상위·모듈) — 관리자 또는 "프로젝트 생성" 권한 | 유형·상태·사용자 정의 필드 **새로 만들기** |
| 이미 있는 유형(`tracker_ids`)·사용자 정의 필드(`issue_custom_field_ids`) 켜기 | 워크플로 |
| 목표버전(`/projects/<id>/versions.json`), 범주(`issue_categories.json`) | 우선순위 값, 역할과 권한 |
| 구성원(그 Redmine에 계정이 있는 사용자만) | REST API 켜기, 첨부 크기 등 전체 설정 |
- 사용자 정의 필드 정의(`/custom_fields.json`)와 그 선택값 목록은 관리자 키로만 조회된다. 원본 설정을 일감 데이터에서만 모으면 한 번도 쓰이지 않은 유형·버전은 알 수 없다.
- 실용적인 방식은 반자동이다: 원본 설정을 읽어 대상에 없는 것을 "관리자 할 일" 목록으로 보여 주고, API로 되는 것만 만든다. 다시 실행하면 빠진 것만 더하게(멱등) 만든다. 식별자는 만든 뒤 바꿀 수 없으니 미리 보기 단계에서 형식·중복을 검사한다.

## 핵심 코드
### 1) 상세와 첨부 받기 (PowerShell 5.1)
```powershell
$base = 'https://redmine.example.com'; $key = Read-Host 'API key'     # 키는 파일에 저장하지 않는다
$h = @{ 'X-Redmine-API-Key' = $key }
$issue = (Invoke-RestMethod "$base/issues/1234.json?include=journals,attachments" -Headers $h -TimeoutSec 60).issue
foreach ($a in $issue.attachments) {
  Invoke-WebRequest $a.content_url -Headers $h -OutFile (Join-Path $outDir $a.filename) -TimeoutSec 120
}
```

### 2) 조건 검색을 100건씩 나눠 받기
```powershell
$q = 'project_id=myproj&status_id=open&assigned_to_id=me&subject=~keyword'   # 화면 주소의 쿼리도 그대로 가능
$all = @(); $offset = 0
do {
  $r = Invoke-RestMethod "$base/issues.json?$q&limit=100&offset=$offset" -Headers $h
  $all += $r.issues; $offset += 100
} while ($offset -lt $r.total_count -and $offset -lt 1000)
```

### 3) 상태 변경 후 다시 조회해 확인
```powershell
$validIds = (Invoke-RestMethod "$base/issue_statuses.json" -Headers $h).issue_statuses.id
if ($validIds -notcontains $toStatusId) { throw "unknown status id $toStatusId" }   # 잘못된 값은 보내지 않는다
$body = @{ issue = @{ status_id = $toStatusId; notes = $note } } | ConvertTo-Json -Depth 3
Invoke-RestMethod "$base/issues/$id.json" -Method Put -Headers $h -ContentType 'application/json; charset=utf-8' `
  -Body ([Text.Encoding]::UTF8.GetBytes($body))
$now = (Invoke-RestMethod "$base/issues/$id.json" -Headers $h).issue.status.id
if ($now -ne $toStatusId) { "not changed (workflow?)" }
```

### 4) 변경 요청 파일 예
```json
{ "version": 1, "createdAt": "2026-01-01T10:00:00",
  "items": [ { "id": 1234,
    "seen":   { "statusId": 2, "assignee": "user-a" },
    "change": { "statusId": 3, "assigneeId": 45, "assigneeName": "user-b", "notes": "patched" } } ] }
```

### 5) 비공개 댓글은 권한 확인 뒤에만, 올린 뒤 재확인
```powershell
$me = (Invoke-RestMethod "$base/users/current.json?include=memberships" -Headers $h).user
$roleIds = ($me.memberships | Where-Object { $_.project.id -eq $projectId }).roles.id
$perms = foreach ($r in $roleIds) { (Invoke-RestMethod "$base/roles/$r.json" -Headers $h).role.permissions }
$canPrivate = $me.admin -or ($perms -contains 'set_notes_private')   # 확인 실패(예외)면 $false로 둔다
if ($canPrivate) {
  $body = @{ issue = @{ notes = $privateText; private_notes = $true } } | ConvertTo-Json -Depth 3
  Invoke-RestMethod "$base/issues/$id.json" -Method Put -Headers $h -ContentType 'application/json; charset=utf-8' `
    -Body ([Text.Encoding]::UTF8.GetBytes($body))
  $last = ((Invoke-RestMethod "$base/issues/$id.json?include=journals" -Headers $h).issue.journals)[-1]
  if (-not $last.private_notes) { "WARN: posted as public note" }
} else { "skip private note (no permission)" }
```

### 6) 프로젝트 만들고 기존 유형 켜기
```powershell
$trackerIds = (Invoke-RestMethod "$base/trackers.json" -Headers $h).trackers |
  Where-Object { $wantNames -contains $_.name } | ForEach-Object { $_.id }       # 이름이 같은 것만
$p = @{ project = @{ name = 'Copy project'; identifier = 'copy-proj'; is_public = $false
        enabled_module_names = @('issue_tracking'); tracker_ids = @($trackerIds) } } | ConvertTo-Json -Depth 4
Invoke-RestMethod "$base/projects.json" -Method Post -Headers $h -ContentType 'application/json; charset=utf-8' `
  -Body ([Text.Encoding]::UTF8.GetBytes($p))                                     # 권한 없으면 403
$v = @{ version = @{ name = 'v1.0' } } | ConvertTo-Json
Invoke-RestMethod "$base/projects/copy-proj/versions.json" -Method Post -Headers $h -ContentType 'application/json' -Body $v
```

## 주의할 점
- 비공개 댓글을 복제할 계획이면 **복제 전에** 권한부터 준다. 권한 없이 한 번 돌려 "옮긴 이력"으로 기록되면 나중에 권한을 받아도 내용이 다시 채워지지 않는다.
- 고객 데이터를 망 밖으로 가져오는 일이면 반출 규정을 먼저 확인하고, 다 본 사본은 지운다. 스크린샷에 개인정보가 있을 수 있다.
- PowerShell 5.1에서 한글 본문을 PUT할 때는 문자열이 아니라 UTF-8 바이트로 보낸다.
- 시험은 가짜 서버나 시험 프로젝트로 한다. 실제 일감에 잘못된 요청이 한 번 가면 수정 시각 같은 흔적은 되돌릴 수 없다.
- 사설 인증서 사이트는 인증서 검사 생략 옵션이 필요할 수 있다(그 사이트에만 한정).

## 참고
- Redmine 문서: Rest API, Rest Issues, Rest Queries
