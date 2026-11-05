# MariaDB 뷰 DEFINER 계정과 콜레이션 오류(1449·1267)

> 분류: DB·SQL · 최근 갱신: 2026-11-04

## 한 줄 요약
다른 서버의 덤프를 들여온 MariaDB/MySQL에서 뷰 조회가 `ERROR 1449`(DEFINER 계정 없음)나 `ERROR 1267`(Illegal mix of collations)로 실패할 수 있다. 앞의 것은 계정 이름 대소문자 차이, 뒤의 것은 뷰 안 문자 상수가 만들 때의 접속 콜레이션으로 저장되기 때문이다. 앱에서는 이 오류가 NPE 한 줄로만 보일 수 있다.

## 내용
### ERROR 1449: The user specified as a definer does not exist
- 뷰는 `SQL SECURITY DEFINER`가 기본이라, 조회할 때 뷰를 만든 계정(DEFINER)의 권한으로 실행한다. 그 계정이 없으면 뷰를 쓸 수 없다.
- MariaDB/MySQL 계정 이름은 **대소문자를 구분**한다. 덤프의 DEFINER가 `appuser@%`이고 접속 계정이 `APPUSER@%`이면 "없는 계정"이다.
- 해결 방법 세 가지
  - 소문자 계정을 하나 더 만들고 잠가 둔다(`ACCOUNT LOCK`). 접속 설정을 건드리지 않고, 다음에 덤프를 다시 들여와도 맞는다. 권장.
  - 뷰를 `ALTER DEFINER=CURRENT_USER`로 다시 만든다. 덤프를 다시 들여오면 원래대로 돌아간다.
  - 접속 계정 이름을 바꾼다(`RENAME USER`). 앱 접속 설정도 같이 바꿔야 해서 위험하다.

### ERROR 1267: Illegal mix of collations (…,COERCIBLE) and (…,COERCIBLE)
- 뷰의 계산식 안 문자 상수(`CASE WHEN ... THEN '002' END`)는 **뷰를 만들 때 세션의 콜레이션**으로 저장된다.
- 앱 JDBC 연결의 콜레이션이 다르면(예: 뷰는 `utf8mb4_general_ci`, 드라이버 기본은 `utf8mb4_unicode_ci`) 쿼리의 상수와 뷰의 상수를 비교할 때 둘 다 "강제력 같음(COERCIBLE)"이라 어느 쪽을 따를지 못 정하고 오류를 낸다.
- mysql 클라이언트로 같은 SQL을 돌리면 클라이언트 세션 콜레이션이 달라 **정상으로 나와 원인을 놓치기 쉽다.** `SET NAMES utf8mb4 COLLATE <앱과 같은 콜레이션>` 뒤에 실행해 재현한다.
- 해결: 앱과 같은 콜레이션 세션에서 뷰를 정의 그대로 다시 만든다. 백업 먼저.

### 앱에서는 NPE로만 보이는 경우
- Spring의 `SQLExceptionTranslator`가 모르는 오류 코드에 null을 돌려주고, 템플릿 코드가 그 결과를 그대로 `throw`하면 `throw null` → `NullPointerException`이 된다. 원래 SQL 오류 메시지가 사라진다.
- JVM은 같은 예외가 반복되면 스택을 생략하므로(OmitStackTraceInFastThrow) 콘솔에는 NPE 한 줄만 남는다.
- 이럴 때는 매퍼의 SQL을 펼쳐 **앱과 같은 세션 조건**에서 직접 실행하거나, 같은 클래스패스로 매퍼만 띄워 실행해 실제 SQL 오류를 본다.

## 핵심 코드
### 1) 뷰 DEFINER·콜레이션 확인
```sql
SELECT TABLE_NAME, DEFINER, SECURITY_TYPE, CHARACTER_SET_CLIENT, COLLATION_CONNECTION
  FROM information_schema.VIEWS
 WHERE TABLE_SCHEMA = DATABASE();
SELECT user, host FROM mysql.user WHERE user IN ('appuser', 'APPUSER');
```

### 2) 1449: 잠긴 DEFINER 계정 추가
```sql
CREATE USER 'appuser'@'%' ACCOUNT LOCK;            -- 로그인은 불가, 뷰 권한 확인용
GRANT ALL PRIVILEGES ON mydb.* TO 'appuser'@'%';
```

### 3) 1267: 재현과 뷰 재생성
```sql
SET NAMES utf8mb4 COLLATE utf8mb4_unicode_ci;     -- 앱 연결과 같게
SELECT COUNT(*) FROM v_pending_docs WHERE func_type <> '004';   -- 여기서 1267 재현

SHOW CREATE VIEW v_pending_docs;                   -- 정의를 백업해 두고
CREATE OR REPLACE ALGORITHM=UNDEFINED DEFINER=`appuser`@`%` SQL SECURITY DEFINER
VIEW v_pending_docs AS
SELECT doc_id,
       CASE WHEN state = '000' AND func_type <> '004' THEN '002' ELSE func_type END AS func_type
  FROM docs;                                       -- 같은 세션에서 다시 만들면 상수가 unicode_ci
```

### 4) 덤프 백업 (뷰만)
```bash
mysqldump --no-data --skip-triggers mydb v_pending_docs v_users > views-backup.sql
```

## 주의할 점
- 개발 PC·테스트 DB에서만 생기는 환경 문제인지 먼저 가린다. 운영이 같은 콜레이션으로 뷰를 만들었다면 운영에는 없다.
- 뷰를 다시 만들 때 DEFINER와 SQL SECURITY를 원래 값으로 유지한다. 바꾸면 권한 동작이 달라진다.
- 콜레이션이 섞인 DB는 조인·비교 성능 문제도 함께 있을 수 있다([MySQL 조인 컬럼 문자셋 불일치로 인덱스를 못 쓰는 문제](MySQL 조인 컬럼 문자셋 불일치로 인덱스를 못 쓰는 문제.md)).

## 참고
- MariaDB Knowledge Base: CREATE VIEW (DEFINER, SQL SECURITY), Coercibility
- MySQL Reference Manual: Collation Coercibility in Expressions, Error 1267·1449
