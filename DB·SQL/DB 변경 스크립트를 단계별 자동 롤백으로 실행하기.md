# DB 변경 스크립트를 단계별 자동 롤백으로 실행하기

> 분류: DB·SQL · 최근 갱신: 2026-11-19

## 한 줄 요약
MySQL·MariaDB의 DDL은 암묵적으로 커밋되어 트랜잭션으로 되돌릴 수 없다. 여러 DDL·DML을 공용 DB에 넣을 때는 단계마다 "이미 있으면 건너뜀 → 실행 전 상태 기록 → 실행 → 되돌리기 문장 쌓기"를 하고, 실패하면 앞 단계를 역순으로 되돌리는 실행기를 쓰면 안전하다.

## 내용
- 단계 구성: 테이블 생성, 컬럼 추가, 인덱스 추가, 문자셋 변환, 설정 행 INSERT, 값 UPDATE를 한 단계씩 나눈다.
- 각 단계 전에 존재 확인(`information_schema`)을 해서 이미 있으면 실행하지 않는다. 그래서 다시 돌려도 된다.
- 되돌리기 정보를 실행 전에 남긴다.
  - 만든 테이블·컬럼·인덱스 → DROP 문장
  - 넣은 행 → 키로 DELETE
  - UPDATE → 바꾸기 전 값을 행마다 읽어 원래 값으로 UPDATE
  - 문자셋 변환·컬럼 정의 변경 → `SHOW CREATE TABLE` 원문 저장
- 실패하면 실패한 단계(DML)는 트랜잭션 롤백, 이미 성공한 단계는 쌓아 둔 되돌리기 문장을 역순 실행한다. 실행 로그와 `rollback.sql` 파일을 남겨 나중에 수동으로도 되돌릴 수 있게 한다.
- 실제로 걸린 실패 두 가지:
  1. **외래 키**: 코드 데이터에는 있지만 부모 테이블에는 없는 키(예: 존재하지 않는 테넌트 번호)로 설정 행을 넣다가 FK 오류. 넣을 대상 키가 부모 테이블에 있는지 먼저 확인한다.
  2. **AUTO_INCREMENT 컬럼에 0 넣기**: MySQL은 AUTO_INCREMENT 컬럼에 0을 넣으면 "다음 번호를 달라"로 해석해 1 등으로 바꾼다. 0번 행을 가진 초기 데이터를 넣으면 다음 행과 키가 겹친다. 원본 초기 데이터 스크립트가 `SQL_MODE='NO_AUTO_VALUE_ON_ZERO'`를 걸어 두는 경우가 많은데, INSERT 문만 뽑아 옮기면 이 설정이 빠진다.
- 큰 테이블의 인덱스 추가·문자셋 변환은 테이블 재작성이라 그동안 쓰기가 막힐 수 있다. 사용자가 적을 때 실행한다.

## 핵심 코드
```sql
-- 0번 행을 그대로 넣기(세션 범위). 끝나면 원래대로
SET @OLD_SQL_MODE = @@SESSION.sql_mode;
SET SESSION sql_mode = CONCAT(@@SESSION.sql_mode, ',NO_AUTO_VALUE_ON_ZERO');
INSERT INTO timezones (timezone_id, name) VALUES (0, 'UTC');
SET SESSION sql_mode = @OLD_SQL_MODE;
```

```sql
-- 단계 전 존재 확인 예
SELECT COUNT(*) FROM information_schema.STATISTICS
 WHERE TABLE_SCHEMA = DATABASE() AND TABLE_NAME = 'orders' AND INDEX_NAME = 'idx_orders_user';
-- 0이면 실행, 되돌리기 문장으로 'DROP INDEX idx_orders_user ON orders' 기록
```

```java
// 실행기 뼈대: 성공한 단계의 되돌리기 문장을 쌓고 실패 시 역순 실행
Deque<String> undo = new ArrayDeque<>();
try {
    for (Step s : plan) {
        if (s.alreadyApplied(conn)) continue;
        String rollbackSql = s.captureUndo(conn);   // 실행 전 값·SHOW CREATE TABLE 기록
        s.execute(conn);
        undo.push(rollbackSql);
        log.write(rollbackSql);                     // rollback.sql 파일에도 남김
    }
} catch (SQLException e) {
    conn.rollback();                                // 실패한 DML 단계만 해당
    while (!undo.isEmpty()) exec(conn, undo.pop()); // 앞 단계는 역순으로 되돌림
    throw e;
}
```

## 주의할 점
- UPDATE 문법만 보고 싶을 때는 `EXPLAIN UPDATE ...`로 데이터 변경 없이 확인할 수 있다(MariaDB·MySQL 5.6+).
- 되돌리기가 데이터를 지우는 단계(새로 만든 테이블 DROP)는 그사이 들어온 데이터도 함께 사라진다. 운영 중에는 실패 즉시 되돌리는 범위를 사람이 정한다.
- 여러 서버에 넣을 때는 변경이 적은 서버부터 하고, 성공했을 때만 다음 서버로 간다.
- 관련: [AUTO_INCREMENT가 건너뛰며 증가하는 이유](AUTO_INCREMENT가 건너뛰며 증가하는 이유.md), [인덱스를 타는 조건과 온라인 인덱스 추가(INPLACE·LOCK=NONE)](인덱스를 타는 조건과 온라인 인덱스 추가(INPLACE·LOCK=NONE).md)

## 참고
- MySQL 문서: Statements That Cause an Implicit Commit, Server SQL Modes(NO_AUTO_VALUE_ON_ZERO)
