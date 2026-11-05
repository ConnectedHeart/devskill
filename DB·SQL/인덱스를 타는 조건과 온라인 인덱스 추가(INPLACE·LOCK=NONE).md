# 인덱스를 타는 조건과 온라인 인덱스 추가(INPLACE·LOCK=NONE)

> 분류: DB·SQL · 최근 갱신: 2026-11-04

## 한 줄 요약
인덱스 컬럼을 함수로 감싼 조건은 인덱스를 못 쓰므로 같은 의미의 범위 조건으로 바꾸고, 앞 컬럼부터 맞춰 찾는 복합 인덱스는 쿼리 용도마다 따로 둔다. 운영 MariaDB/MySQL에 인덱스를 넣을 때는 `ALGORITHM=INPLACE, LOCK=NONE`을 적어 잠금 방식으로 조용히 바뀌는 것을 막고, Galera에서는 그래도 쓰기가 멈추니 업무 외 시간에 한다.

## 내용
### 함수로 감싼 컬럼은 인덱스를 못 쓴다
- `DATEDIFF(NOW(), created_date) > 10`, `DATE(created_date) = ...`처럼 컬럼에 함수를 씌우면 행마다 계산해야 해서 전체를 훑는다. DELETE라면 훑은 행에 잠금까지 걸린다.
- 같은 의미를 `created_date < (계산한 기준 시각)`으로 바꾸면 범위 검색이 된다. 계산은 상수 쪽(파라미터)에서 한다.
- 바꾼 식이 경계값(자정, 시간대, 보관 일수)에서 정말 같은지 **많은 조합으로 대조**한다. 예: 가상 등록시각 수천 개 × 기준시각(자정 직전·직후 포함) × 보관일 여러 개를 DB에서 두 식으로 계산해 불일치 0인지 본다.
- 컬럼이 UTC로 저장되면 `NOW()`(세션 시간대)를 쓰지 말고 원래 식과 같은 기준 값을 쓴다.

### 용도별 복합 인덱스
- 복합 인덱스는 앞 컬럼부터 차례로 맞춰 찾는다. 조건에 없는 컬럼이 중간에 있으면 그 앞까지만 쓴다.
- 사용자별 개수 조회 `(tenant, user, company, created)`와 기간 정리 `(tenant, created)`처럼 조건이 다르면 인덱스 두 개가 맞다. 하나로 합치면 한쪽 쿼리는 거의 전체를 훑는다.
- 범위 조건(`<>`, `<`, `BETWEEN`) 뒤에 둔 컬럼은 검색에 쓰이지 않는다. 등호 조건 컬럼을 앞에 둔다.
- PK 4번째 컬럼으로만 찾는 쿼리가 많으면(예: `doc_id`) `(tenant, company, doc_id)` 같은 보조 인덱스가 필요하다. UPDATE가 다른 단일 인덱스로 넓게 훑으면 REPEATABLE READ에서 훑은 행을 모두 잠가 같은 범위의 동시 처리끼리 기다린다 — 읽기 속도보다 잠금 범위가 더 큰 이득일 수 있다.

### 온라인 인덱스 추가
- `ALGORITHM=INPLACE`: 테이블을 복사하지 않고 그 자리에서 인덱스만 만든다(COPY는 전체 행을 새 테이블로 옮겨 오래 걸리고 디스크를 두 배로 쓴다).
- `LOCK=NONE`: 만드는 동안 읽기·쓰기를 허용한다.
- 둘을 적는 진짜 이유는 안전장치다. 안 적으면 상황에 따라 잠그는 방식으로 조용히 실행될 수 있고, 적으면 그렇게 못 할 때 오류를 내고 아무것도 하지 않는다.
- 그래도 시작·끝에 짧은 메타데이터 잠금을 잡는다. 그 테이블에 긴 트랜잭션이 열려 있으면 DDL이 기다리고, 뒤에 오는 조회도 줄줄이 막힌다.
- **Galera(TOI 방식)**: DDL은 모든 노드에서 같은 순서로 실행되며 그동안 클러스터 전체 쓰기가 멈춘다. `LOCK=NONE`이 막아 주지 못한다. `wsrep_OSU_method`를 확인한다.
- 행 형식이 COMPACT면 인덱스 컬럼당 767바이트 한도가 있어 긴 varchar(utf8mb4)는 접두 인덱스가 필요하다. `longtext`는 접두 길이 없이 인덱스를 만들 수 없다(1170 오류).

## 핵심 코드
### 1) 함수 조건 → 범위 조건 (MySQL / Oracle)
```sql
-- 전: 인덱스 불가
DELETE FROM notifications
 WHERE tenant_id = ? AND DATEDIFF(?, created_date) > ?;
-- 후: 기준 시각을 상수로 계산 (MySQL)
DELETE FROM notifications
 WHERE tenant_id = ?
   AND created_date < DATE_SUB(DATE(?), INTERVAL ? DAY);
-- Oracle
DELETE FROM notifications
 WHERE tenant_id = ?
   AND created_date < TRUNC(TO_DATE(?, 'YYYY-MM-DD HH24:MI:SS')) - ?;
```

### 2) 두 식이 같은지 DB에서 대조 (MariaDB)
```sql
WITH RECURSIVE n AS (SELECT 0 AS i UNION ALL SELECT i + 1 FROM n WHERE i < 9999)
SELECT COUNT(*) AS mismatch
  FROM n
  CROSS JOIN (SELECT '2026-01-10 23:59:59' AS base UNION ALL SELECT '2026-01-11 00:00:00') b
 WHERE (DATEDIFF(b.base, TIMESTAMP('2026-01-01') + INTERVAL n.i * 97 MINUTE) > 10)
    <> (TIMESTAMP('2026-01-01') + INTERVAL n.i * 97 MINUTE < DATE_SUB(DATE(b.base), INTERVAL 10 DAY));
```

### 3) 운영 적용 스크립트 골격
```sql
SHOW VARIABLES LIKE 'wsrep_OSU_method';        -- TOI면 업무 외 시간
SELECT ROW_FORMAT, TABLE_ROWS FROM information_schema.TABLES
 WHERE TABLE_SCHEMA = DATABASE() AND TABLE_NAME = 'notifications';
SHOW INDEX FROM notifications;                 -- 같은 컬럼 인덱스가 다른 이름으로 있는지

ALTER TABLE notifications
  ADD INDEX IF NOT EXISTS idx_noti_user (tenant_id, user_id, company_id, created_date),
  ADD INDEX IF NOT EXISTS idx_noti_created (tenant_id, created_date),
  ALGORITHM=INPLACE, LOCK=NONE;
ANALYZE TABLE notifications;

EXPLAIN SELECT COUNT(*) FROM notifications
 WHERE tenant_id = 1 AND user_id = 'u1' AND company_id = 'c1' AND created_date > NOW() - INTERVAL 1 DAY;
-- 되돌리기: ALTER TABLE notifications DROP INDEX idx_noti_user, DROP INDEX idx_noti_created;
```

## 주의할 점
- 인덱스는 결과를 바꾸지 않으므로 화면이 아니라 실행 계획(`type`이 `ALL` → `range`/`ref`, rows)으로 효과를 확인한다.
- 인덱스는 INSERT마다 갱신되므로 쓰기가 많은 테이블은 꼭 필요한 것만 둔다.
- 운영 DB에는 이미 같은 인덱스가 다른 이름으로 있을 수 있다. `SHOW INDEX`로 먼저 확인하고 `IF NOT EXISTS`를 쓴다.
- 신규 설치용 DDL 파일에 넣는 것과 운영 DB에 적용하는 것은 따로다. 운영용 SQL을 별도로 준비한다.

## 참고
- MariaDB Knowledge Base: ALTER TABLE (ALGORITHM, LOCK), InnoDB Online DDL
- MariaDB Knowledge Base: Galera Cluster Schema Upgrades (TOI, RSU)
- MySQL Reference Manual: How MySQL Uses Indexes, Multiple-Column Indexes
