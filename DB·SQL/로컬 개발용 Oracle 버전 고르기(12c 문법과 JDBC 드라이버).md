# 로컬 개발용 Oracle 버전 고르기(12c 문법과 JDBC 드라이버)

> 분류: DB·SQL · 최근 갱신: 2026-11-24

## 한 줄 요약
로컬에 시험용 Oracle을 깔 때는 소스가 쓰는 SQL 문법의 최소 버전과 앱의 JDBC 드라이버 세대를 먼저 보고 고른다. 12c 문법을 쓰는 소스에 11g XE는 맞지 않고, 오래된 드라이버로 최신 23ai에 붙으면 인증 방식 오류가 날 수 있다.

## 내용
### 소스에서 최소 버전 찾기
| 문법 | 필요한 버전 |
|---|---|
| `FETCH FIRST n ROWS ONLY`, `OFFSET n ROWS` | 12c 이상 |
| `GENERATED ... AS IDENTITY` (자동 증가 컬럼) | 12c 이상 |
| `LISTAGG`, `REGEXP_LIKE` | 11gR2 이상 |

- 매퍼·DDL에 위 12c 문법이 한 곳이라도 있으면 11g XE로는 그 쿼리가 실패한다.

### 드라이버 세대
- 앱이 쓰는 `ojdbc*.jar`의 이름과 MANIFEST의 Implementation-Version을 본다(빌드 설정의 의존성이 주석 처리되고 lib 폴더 jar를 직접 쓰는 경우가 많다).
- 오래된 드라이버(예: 12.1의 ojdbc7)는 21c에는 대체로 붙지만, 23ai는 공식 지원 조합이 아니어서 `ORA-28040: No matching authentication protocol`이 날 수 있다. 그러면 서버 `sqlnet.ora`의 허용 인증 버전을 낮추거나 드라이버를 올려야 한다.

### 무료판 선택
- **21c XE**: Windows 설치 파일이 있고 무료. CPU 2스레드, 메모리 2GB, 사용자 데이터 12GB 한도. 빈 스키마로 기능 시험에는 충분하다.
- **23ai Free**: 최신이지만 위 드라이버 문제를 확인해야 한다.
- 컨테이너 이미지가 가장 간편하지만 Windows에서는 Docker/WSL이 있어야 한다.
- 개발 PC 메모리가 빠듯하면 서비스를 수동 시작으로 두고 시험할 때만 켠다. 설치에 약 8~10GB 디스크가 든다.

### 설치 뒤
- 저장소의 Oracle DDL·초기 데이터 스크립트로 빈 스키마를 만든다. 다른 DB(예: MariaDB)의 시험 데이터는 옮겨지지 않는다.
- XE 21c는 기본이 CDB 구조라 앱은 PDB 서비스명(`XEPDB1`)으로 접속해야 한다([Oracle 접속 오류로 서비스명과 계정 문제 구분(ORA-12514·01017)](Oracle 접속 오류로 서비스명과 계정 문제 구분(ORA-12514·01017).md)).

## 핵심 코드
### 1) 소스에서 12c 문법·드라이버 버전 찾기
```bash
git grep -nE 'FETCH FIRST|OFFSET [^ ]+ ROWS' -- '<매퍼 경로>'
git grep -nE 'GENERATED .*AS IDENTITY' -- '<DDL 경로>' '<매퍼 경로>'
unzip -p <lib 경로>/ojdbc*.jar META-INF/MANIFEST.MF | grep -i version
```

### 2) 21c XE에 시험용 계정 만들고 스키마 넣기
```sql
-- sqlplus / as sysdba
ALTER SESSION SET CONTAINER = XEPDB1;
CREATE USER app_dev IDENTIFIED BY "<비밀번호>" DEFAULT TABLESPACE USERS QUOTA UNLIMITED ON USERS;
GRANT CREATE SESSION, CREATE TABLE, CREATE VIEW, CREATE SEQUENCE, CREATE PROCEDURE, CREATE TRIGGER TO app_dev;
-- 이후: sqlplus app_dev/<비밀번호>@localhost:1521/XEPDB1 @<DDL 스크립트>.sql
```

### 3) 앱 접속 URL과 서비스 수동 시작
```properties
jdbc.url = jdbc:oracle:thin:@//localhost:1521/XEPDB1
```
```powershell
Set-Service OracleServiceXE -StartupType Manual   # 평소엔 꺼 두기
Start-Service OracleServiceXE                     # 시험할 때만
```

### 4) 오래된 드라이버로 ORA-28040이 날 때 (서버 sqlnet.ora)
```text
SQLNET.ALLOWED_LOGON_VERSION_SERVER = 11
```

## 주의할 점
- 로컬 DB 종류가 운영과 다르면 다른 DB용 매퍼 파일의 문법 오류·중복 ID 같은 문제가 로컬 기동에서 드러나지 않는다. 시험용 Oracle은 이런 오류를 잡는 데 의미가 있다.
- 인증 버전을 낮추는 설정은 로컬 시험용에만 쓰고, 운영은 드라이버를 올려 해결한다.
- 내려받기에 오라클 계정 로그인이 필요하다.

## 참고
- Oracle Database Express Edition(XE) 21c 문서, Oracle Database 23ai Free
- Oracle JDBC 드라이버와 DB 버전 호환표(JDBC FAQ)
