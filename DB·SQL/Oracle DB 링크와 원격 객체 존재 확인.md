# Oracle DB 링크와 원격 객체 존재 확인

> 분류: DB·SQL · 최근 갱신: 2026-10-27

## 한 줄 요약
`테이블@링크` 형태로 다른 DB의 객체를 쓰는 코드가 있을 때, 링크 존재 → 접속 → 원격 객체 존재 → 실제 읽기 권한 순으로 나눠 확인하면 어느 단계에서 막혔는지 오류 코드로 바로 가를 수 있다.

## 내용
- **DB 링크**: 한 Oracle DB에서 다른 DB의 객체를 `owner.object@link_name`으로 부를 수 있게 하는 접속 정의다. 링크에는 원격 접속 주소와 원격 계정이 들어 있고, 원격 쪽에서는 그 계정의 권한으로 동작한다. Tibero도 같은 문법과 사전 뷰를 쓴다.
- **링크가 있는지**: `ALL_DB_LINKS`에서 찾는다. 이름 뒤에 도메인이 붙을 수 있으므로(`MYLINK.WORLD`) `LIKE`로 찾는다. OWNER가 `PUBLIC`이면 모든 계정이 쓸 수 있는 공용 링크다. DBA 권한이 있으면 `DBA_DB_LINKS`로 전체를 본다.
- **접속이 되는지**: `DUAL@링크`를 한 번 조회해 본다. 오류 코드로 원인이 갈린다.

  | 오류 | 의미 |
  |---|---|
  | `ORA-02019` | 그런 이름의 링크가 없음 |
  | `ORA-12154` / `ORA-12541` | 접속 주소(TNS 별칭·리스너) 문제 |
  | `ORA-01017` | 링크에 든 원격 계정·비밀번호 문제 |

- **원격 객체가 있는지**: `ALL_OBJECTS@링크`로 원격 사전을 본다. 테이블만이 아니라 뷰·프로시저·함수도 나오므로 `OBJECT_TYPE`을 같이 본다. `ALL_*` 뷰는 **링크 계정이 권한을 가진 객체만** 보여 주므로, 결과가 없다고 객체가 없다고 단정하지 않는다.
- **실제로 읽히는지**: 한 행만 조회해 본다. `ORA-00942`(테이블 또는 뷰가 존재하지 않음)는 객체가 없거나 링크 계정에 SELECT 권한이 없는 경우 모두 나온다.
- **시노님**: 코드가 링크를 직접 쓰지 않고 시노님을 거칠 수도 있다. `ALL_SYNONYMS`의 `DB_LINK` 컬럼으로 링크를 쓰는 시노님을 찾는다.

## 핵심 코드
### 1) 단계별 확인 쿼리
```sql
-- 1. 링크 존재
SELECT OWNER, DB_LINK, USERNAME, HOST, CREATED
  FROM ALL_DB_LINKS
 WHERE DB_LINK LIKE 'MYLINK%';

-- 2. 접속
SELECT SYSDATE FROM DUAL@MYLINK;

-- 3. 원격 객체 존재
SELECT OWNER, OBJECT_NAME, OBJECT_TYPE, STATUS
  FROM ALL_OBJECTS@MYLINK
 WHERE OWNER = 'REMOTE_OWNER'
   AND OBJECT_NAME = 'MY_VIEW';

-- 4. 실제 읽기
SELECT * FROM REMOTE_OWNER.MY_VIEW@MYLINK WHERE ROWNUM = 1;

-- 5. 링크를 쓰는 시노님
SELECT OWNER, SYNONYM_NAME, TABLE_OWNER, TABLE_NAME, DB_LINK
  FROM ALL_SYNONYMS
 WHERE DB_LINK LIKE 'MYLINK%';

-- 끝나면 링크 세션 닫기
COMMIT;
ALTER SESSION CLOSE DATABASE LINK MYLINK;
```

### 2) ORA-00942일 때: 링크 계정과 권한 확인
```sql
-- 링크가 원격에서 어떤 계정으로 접속하는지
SELECT USER FROM DUAL@MYLINK;

-- 그 계정(또는 PUBLIC·활성 롤)이 원격 객체에 받은 권한
SELECT GRANTEE, TABLE_SCHEMA, TABLE_NAME, PRIVILEGE
  FROM ALL_TAB_PRIVS@MYLINK
 WHERE TABLE_SCHEMA = 'REMOTE_OWNER'
   AND TABLE_NAME = 'MY_VIEW';
```

### 3) 링크를 쓰는 곳 찾기
```sql
-- DB 안의 프로시저·함수·패키지 소스
SELECT OWNER, NAME, TYPE, LINE
  FROM ALL_SOURCE
 WHERE UPPER(TEXT) LIKE '%@MYLINK%';
```
```bash
# 애플리케이션 소스·매퍼
grep -rni "@MYLINK" src/
```

### 4) (테스트 환경) 링크 만들기
```sql
-- CREATE DATABASE LINK 권한 필요. 비밀번호는 결과 공유·로그에 남기지 않는다
CREATE DATABASE LINK MYLINK
  CONNECT TO remote_user IDENTIFIED BY "<비밀번호>"
  USING '<TNS 별칭>';
```

## 주의할 점
- 조회만 해도 링크를 쓰면 원격 세션이 열리고 분산 트랜잭션이 시작된 상태가 된다. `ALTER SESSION CLOSE DATABASE LINK` 전에 `COMMIT`(또는 `ROLLBACK`)해야 닫힌다.
- 객체 이름은 사전에 대문자로 저장되므로 비교 값도 대문자로 쓴다.
- 링크 계정의 비밀번호나 접속 정보를 결과 공유 시 그대로 옮기지 않는다. `HOST` 컬럼에 접속 주소가 나올 수 있다.

## 참고
- Oracle Database Administrator's Guide — Managing Database Links
- Oracle Database Reference — ALL_DB_LINKS, ALL_OBJECTS, ALL_SYNONYMS
