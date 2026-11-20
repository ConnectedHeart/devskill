# 폐쇄망 빌드용 Maven 로컬 저장소 파일 골라 보내기

> 분류: Java·Spring · 최근 갱신: 2026-11-19

## 한 줄 요약
인터넷이 안 되는 빌드 PC에 새 의존성을 넘길 때는, 상대 PC와 같은 "변경 전" 상태의 빈 로컬 저장소를 만들어 두고 새 소스를 빌드할 때 추가로 내려받힌 파일만 모아 같은 폴더 구조로 보낸다. 그 폴더만 더해 `mvn -o`(오프라인) 빌드가 되는지로 검증한다.

## 내용
- pom에 라이브러리 하나를 더해도 그 라이브러리가 끌고 오는 의존성(전이 의존성)과 부모 pom까지 필요하다. 예: 캐시 라이브러리 하나에 annotation jar·부모 pom 여러 개.
- 한 번도 빌드한 적 없는 프로젝트(예: 모바일 모듈)는 의존성뿐 아니라 빌드 플러그인(war·surefire 플러그인과 그 의존성)까지 없을 수 있어 파일 수가 훨씬 많다.
- 고르는 절차:
  1. 빈 임시 로컬 저장소(`-Dmaven.repo.local=<임시>`)로 **변경 전 소스**를 빌드한다 → 상대 PC가 이미 가졌을 상태.
  2. 그 폴더를 복사해 둔 뒤 같은 저장소로 **변경 후 소스**를 빌드한다.
  3. 두 상태의 차이(새로 생긴 파일)만 모은다. jar·pom·sha1을 함께 넣는다.
  4. 1번 상태 + 모은 폴더로 오프라인 빌드를 해서 성공하는지 확인한다.
- `_remote.repositories` 파일은 빼는 편이 낫다. 다른 저장소 ID에서 받은 것으로 기록돼 있으면 Maven이 설정된 저장소와 맞지 않는다고 보고 그 파일을 쓰지 않을 수 있다.
- 받는 쪽은 폴더 내용을 `%USERPROFILE%\.m2\repository`(또는 설정된 로컬 저장소)에 그대로 복사한다. 같은 파일을 덮어써도 문제없다.
- 필수(새 의존성)와 조건부(첫 빌드용) 폴더를 나눠 주면 받는 쪽이 판단하기 쉽다.

## 핵심 코드
```powershell
# 1) 변경 전 소스로 기준 저장소 만들기
mvn -Dmaven.repo.local=C:\tmp\repo-base -DskipTests package -f old\pom.xml
Copy-Item C:\tmp\repo-base C:\tmp\repo-after -Recurse
# 2) 변경 후 소스 빌드(같은 저장소에 추가로 받힘)
mvn -Dmaven.repo.local=C:\tmp\repo-after -DskipTests package -f new\pom.xml
```

```powershell
# 3) 새로 생긴 파일만 같은 상대 경로로 모으기(_remote.repositories 제외)
$base = 'C:\tmp\repo-base'; $after = 'C:\tmp\repo-after'; $out = 'C:\tmp\repo-delta'
Get-ChildItem $after -Recurse -File | Where-Object { $_.Name -ne '_remote.repositories' } | ForEach-Object {
  $rel = $_.FullName.Substring($after.Length)
  if (-not (Test-Path ($base + $rel))) {
    New-Item -ItemType Directory -Force (Split-Path ($out + $rel)) | Out-Null
    Copy-Item $_.FullName ($out + $rel)
  }
}
```

```powershell
# 4) 검증: 기준 + 차이분만으로 오프라인 빌드
Copy-Item C:\tmp\repo-base C:\tmp\repo-check -Recurse
Copy-Item C:\tmp\repo-delta\* C:\tmp\repo-check -Recurse -Force
mvn -o -Dmaven.repo.local=C:\tmp\repo-check -DskipTests package -f new\pom.xml
```

## 주의할 점
- 상대 PC의 Maven 버전·settings.xml이 다르면 다른 플러그인 버전을 찾을 수 있다. 오류 메시지의 좌표(groupId:artifactId:version)를 받아 추가로 보낸다.
- 라이브러리 jar가 소스 안(`WEB-INF/lib` 등)에 들어 있는 의존성은 저장소 파일이 필요 없다.
