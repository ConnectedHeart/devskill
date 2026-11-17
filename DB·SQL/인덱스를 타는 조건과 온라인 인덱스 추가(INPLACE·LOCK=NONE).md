# 인덱스를 타는 조건과 온라인 인덱스 추가(INPLACE·LOCK=NONE)

> 분류: DB·SQL · 최근 갱신: 2026-11-16

## 한 줄 요약
인덱스 컬럼을 함수로 감싼 조건은 인덱스를 못 쓰므로 같은 의미의 범위 조건으로 바꾸고, 앞 컬럼부터 맞춰 찾는 복합 인덱스는 쿼리 용도마다 따로 둔다. 운영 MariaDB/MySQL에 인덱스를 넣을 때는 `ALGORITHM=INPLACE, LOCK=NONE`을 적어 잠금 방식으로 조용히 바뀌는 것을 막고, Galera에서는 그래도 쓰기가 멈추니 업무 외 시간에 한다.

## 내용
### 함수로 감싼 컬럼은 인덱스를 못 쓴다
- `DATEDIFF(NOW(), created_date) > 10`, `DATE(created_date) = ...`처럼 컬럼에 함수를 씌우면 행마다 계산해야 해서 전체를 훑는다. DELETE라면 훑은 행에 잠금까지 걸린다.
- 같은 의미를 `created_date < (계산한 기준 시각)`으로 바꾸면 범위 검색이 된다. 계산은 상수 쪽(파라미터)에서 한다.
- 바꾼 식이 경계값(자정, 시간대, 보관 일수)에서 정말 같은지 **많은 조합으로 대조**한다. 예: 가상 등록시각 수천 개 × 기준시각(자정 직전·직후 포함) × 보관일 여러 개를 DB에서 두 식으로 계산해 불일치 0인지 본다.
- 컬럼이 UTC로 저장되면 `NOW()`(세션 시간대)를 쓰지 말고 원래 식과 같은 기준 값을 쓴다.

### 문자 컬럼과 숫자 값 비교도 인덱스를 못 쓴다(암묵적 형변환)
- 문자 컬럼(`target_id VARCHAR`)을 숫자 컬럼(`f.file_id BIGINT`)과 `=`로 비교하면 MySQL/MariaDB는 둘을 숫자로 바꿔 비교한다. 문자 컬럼 쪽이 변환되므로 그 컬럼의 인덱스를 쓰지 못한다. Oracle도 문자 쪽에 `TO_NUMBER`가 씌워져 같다.
- 행마다 `EXISTS (SELECT 1 FROM shares WHERE target_id = f.file_id)`를 검사하는 목록이면, 인덱스를 추가해도 형이 다르면 효과가 작다(실측 1.07초 → 인덱스만 0.64초 → 형까지 맞추면 0.005초).
- 해결: **숫자 쪽을 문자로** 바꿔 비교한다(MySQL `CONCAT(f.file_id, '')`, Oracle `TO_CHAR(f.file_id)`). 인덱스가 있는 컬럼 쪽은 손대지 않는 것이 원칙이다. 문자 쪽을 숫자로 바꾸면 `'007'`과 `7`이 같아지는 등 의미가 달라질 수 있다.
- 파생 테이블(서브쿼리 결과)과 조인할 때도 같다. 파생 테이블이 문자 ID를 내고 바깥이 숫자 ID로 조인하면 옵티마이저가 자동 키를 못 쓴다. 이때는 파생 테이블 쪽 열을 바깥 형으로 맞춰(`CAST(id AS UNSIGNED)`) 내보내면 조인 키를 쓴다(실측 3초 → 0.35초). 값이 모두 숫자 문자열인지 먼저 확인한다.

### 긴 문자 컬럼 하나로 찾는 확인 쿼리(접두 인덱스)
- "이 파일 경로를 쓰는 행이 있나"처럼 PK가 아닌 긴 문자 컬럼 하나로 `COUNT`하는 확인 쿼리는, 인덱스가 없으면 매번 전체를 훑고(EXPLAIN `type=ALL`) 행 수에 비례해 느려진다(실측 115만 행 0.94초 → 인덱스 후 0.0008초). 삭제 대상마다 한 번씩 돌면 그 배수만큼 쌓인다.
- 로컬 데이터가 작아 재현이 안 되면, 같은 데이터를 임시 테이블에 여러 번 복사해 행을 늘려 재 보면 운영 규모의 시간을 추정할 수 있다.
- MySQL/MariaDB InnoDB 인덱스 키 한도는 3072바이트다. `VARCHAR(1020)` utf8mb4(글자당 최대 4바이트)는 전체로 걸 수 없으므로 앞부분만 색인하는 **접두 인덱스** `(col(255))`를 쓴다. 접두가 같은 후보는 DB가 원래 값으로 다시 비교하므로 결과는 틀리지 않고, 경로처럼 앞부분에서 대부분 갈리면 효과도 같다. Oracle은 그 길이 제한 안이면 전체 컬럼으로 만든다.

### 인덱스 만드는 문법(MySQL `ADD KEY`와 `CREATE INDEX`, Oracle)
- MySQL/MariaDB에서 `KEY`와 `INDEX`는 같은 뜻이다. `ALTER TABLE t ADD KEY idx (a, b)`와 `CREATE INDEX idx ON t (a, b)`는 같은 결과다. `ALTER TABLE` 형식은 한 문장에 인덱스 여러 개를 붙여 큰 테이블을 한 번만 다시 쓰게 할 수 있다.
- Oracle·Tibero에는 `ADD KEY` 문법이 없고 일반 인덱스는 `CREATE INDEX`로만 만든다. `ALTER TABLE`로 인덱스가 생기는 것은 PK·UNIQUE 제약을 걸 때뿐이다.
- 이미 있을 때: MySQL "Duplicate key name", Oracle 같은 이름 ORA-00955, **같은 컬럼 조합**의 인덱스가 이미 있으면 ORA-01408(UNIQUE 제약의 인덱스 포함). 그 문장만 실패한다.
- Oracle 함수 인덱스(`TO_NUMBER(col)`)는 숫자로 바꿀 수 없는 값이 한 행이라도 있으면 만들어지지 않는다. 먼저 `TRANSLATE`로 숫자 아닌 글자가 있는 행 수를 센다.

### UNION과 UNION ALL
- `UNION`은 결과 전체의 중복을 지우려고 정렬·해시를 한 번 더 한다. 넓은 행이나 큰 결과에서는 이 비용이 크다.
- 두 분기가 겹칠 수 없을 때(예: 한쪽은 `type='D'`, 다른 쪽은 `type='F'`)는 `UNION ALL`로 바꿔도 결과가 같다. 겹칠 수 있는지 분기 조건으로 먼저 증명한다.

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
### 0) 형이 다른 비교 맞추기와 UNION ALL
```sql
-- 전: 문자 컬럼 = 숫자 → shares 인덱스를 못 씀
SELECT f.file_id, f.name,
       EXISTS (SELECT 1 FROM shares s WHERE s.target_id = f.file_id AND s.target_type = 'F') AS shared
  FROM files f WHERE f.folder_id = ?;
-- 후: 숫자 쪽을 문자로 (MySQL)  /  Oracle은 TO_CHAR(f.file_id)
       EXISTS (SELECT 1 FROM shares s WHERE s.target_id = CONCAT(f.file_id, '') AND s.target_type = 'F') AS shared
CREATE INDEX idx_shares_target ON shares (target_id, target_type, owner_id);

-- 파생 테이블 ID를 바깥 조인 형에 맞추고, 겹칠 수 없는 두 분기는 UNION ALL
SELECT x.* FROM (
    SELECT CAST(s.target_id AS UNSIGNED) AS item_id, 'F' AS kind FROM shares s WHERE s.target_type = 'F' AND s.user_id = ?
    UNION ALL
    SELECT CAST(s.target_id AS UNSIGNED), 'D' FROM shares s WHERE s.target_type = 'D' AND s.user_id = ?
) x JOIN files f ON f.file_id = x.item_id AND x.kind = 'F';
```

### 긴 문자 컬럼 접두 인덱스와 DB별 인덱스 문법
```sql
-- MySQL/MariaDB: VARCHAR(1020) utf8mb4 → 앞 255자만 색인
CREATE INDEX idx_attach_path ON attachments (file_path(255));
-- 같은 뜻 (여러 개를 한 번에 붙이면 테이블 재작성 1회)
ALTER TABLE orders ADD KEY idx_orders_list (tenant_id, created_date, order_id),
                   ADD KEY idx_orders_user (tenant_id, user_id);
EXPLAIN SELECT COUNT(*) FROM attachments WHERE file_path = ?;   -- type=ALL → ref 확인

-- Oracle/Tibero: CREATE INDEX만
CREATE INDEX idx_attach_path ON attachments (file_path);
-- 함수 인덱스 전에 숫자로 못 바꾸는 값이 있는지 (0이어야 함)
SELECT COUNT(*) FROM orders
 WHERE LENGTH(TRIM(TRANSLATE(sort_key, '0123456789', ' '))) > 0;
CREATE INDEX idx_orders_sort ON orders (tenant_id, TO_NUMBER(sort_key));
```

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
