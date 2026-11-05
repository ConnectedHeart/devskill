# MySQL 조인 컬럼 문자셋 불일치로 인덱스를 못 쓰는 문제

> 분류: DB·SQL · 최근 갱신: 2026-11-04

## 한 줄 요약
MySQL·MariaDB에서 조인하는 두 컬럼의 문자셋이 다르면(utf8mb4와 utf8) 한쪽에 변환 함수가 붙어 그 쪽 인덱스를 못 쓴다. 테이블 문자셋을 맞추면 고쳐지고, 데이터가 있어도 안전하게 바꿀 수 있지만 사이트마다 사전 점검이 필요하다.

## 내용
### 왜 느려지나
- `A.item_id = F.item_id`에서 A가 utf8mb4, F가 utf8(utf8mb3)이면 MySQL은 범위가 좁은 쪽(F)을 넓은 쪽으로 변환해 비교한다. 그러면 F의 `item_id` 인덱스를 쓰지 못한다.
- EXPLAIN에서는 F가 `ref PRIMARY key_len=3 ref=const`처럼 앞쪽 컬럼(테넌트 ID 등)까지만 쓰고 rows가 크게 나온다. 바꾼 뒤에는 `key_len`이 커지고 rows=1이 된다.
- 목록 쿼리라면 행마다 변환 쪽 테이블을 크게 훑고, 그다음 GROUP BY → 정렬 → LIMIT으로 처리한다. 그래서 몇 쪽을 보든 비용이 같다.
- 테이블을 오래 운영하다 보면 새로 만든 테이블만 utf8mb4이고 예전 테이블은 utf8인 상태가 흔하다. DDL 파일과 실제 DB(`information_schema.COLUMNS`)를 함께 확인한다.

### 상수 조건이 조인 상대 테이블로 전달되지 않는 경우
- `WHERE a.board_id = ? AND a.board_id = p.board_id`처럼 한쪽에만 상수 조건이 있으면 옵티마이저는 보통 `p.board_id = ?`도 추론해 p를 상수로 한 번 읽는다. 그런데 두 컬럼의 문자셋·콜레이션이 다르면(또는 연결 콜레이션이 `utf8mb4_unicode_ci`이고 컬럼은 `general_ci`면) 이 전달이 일어나지 않아 **행마다 상대 테이블을 조인**하거나 상대 테이블 쪽에서 테넌트 전체를 훑는다.
- mysql 클라이언트 기본 세션에서는 빠르고 앱(JDBC 드라이버 기본 콜레이션)에서만 느려 원인을 놓치기 쉽다. `SET NAMES utf8mb4 COLLATE <앱과 같은 콜레이션>` 뒤에 재 본다.
- 문자셋을 바로 맞출 수 없으면 쿼리로 우회한다: ① 이미 성립하는 조건을 상대 테이블에도 명시(`AND p.board_id = ?`), ② "작은 쪽 테이블에서 먼저 찾고 큰 쪽은 `EXISTS`로 있는지만 확인"으로 바꿔 쓴다. 둘 다 결과 의미는 같다.

### 원인을 확정하는 방법
- 원래 쿼리와, 조인 한쪽에 `CONVERT(A.item_id USING utf8)`를 씌운 변형의 시간·EXPLAIN·결과 해시(MD5)를 비교한다. DDL 없이 효과를 미리 볼 수 있다. 측정 전용이고 코드에 넣지는 않는다.
- 처음 의심한 무거운 부분(윈도 함수, 큰 컬럼 GROUP BY)을 하나씩 뺀 변형도 함께 재서, 줄어드는 비율로 주원인을 가린다.

### 고치는 방법: 테이블 문자셋 맞추기
- `ALTER TABLE t CONVERT TO CHARACTER SET utf8mb4 COLLATE <기준 테이블과 같은 콜레이션>`
- **콜레이션도 같아야 한다.** 문자셋이 같아도 `utf8mb4_general_ci`와 `utf8mb4_unicode_ci`가 섞이면 인덱스를 못 쓰거나 "Illegal mix of collations" 오류가 난다. 고정 문장을 쓰지 말고 기준 테이블의 실제 콜레이션을 읽어 문장을 만든다.
- utf8mb4는 utf8을 포함하므로 데이터가 있어도 잃는 글자가 없다. general_ci끼리라면 비교 결과도 같아 PK 중복이 새로 생기지 않는다.

### 데이터가 있을 때 주의
- 인덱스 컬럼의 문자셋이 바뀌므로 테이블을 통째로 다시 만든다. 그동안 읽기는 되지만 쓰기는 기다린다. 긴 트랜잭션이 있으면 ALTER가 그것을 기다리고 뒤 요청도 줄줄이 막힌다. 테이블 크기만큼 디스크 여유가 필요하다. 업무 외 시간에 한다.
- 인덱스 길이: 행 형식이 COMPACT·REDUNDANT면 **컬럼 하나당** 767바이트, 인덱스 전체는 3072바이트가 한도다. utf8mb4는 글자당 4바이트로 계산한다. 넘으면 `Specified key was too long`으로 실패하고 테이블은 그대로 남는다. DYNAMIC(MySQL 5.7·MariaDB 10.2 이상 기본)이면 컬럼당 3072바이트다.
- 행 크기 65,535바이트 한도, 외래 키, 콜레이션이 바뀔 때의 PK 중복도 미리 본다.

### 통계
- `CONVERT TO`는 행을 다시 넣으므로 `innodb_stats_auto_recalc=ON`이면 통계가 다시 계산된다. 다만 백그라운드라 직후 잠깐은 부정확할 수 있다.
- 인덱스를 새로 만들면 그 인덱스 통계는 생성 시점에 계산된다. 테이블 통계와 다른 인덱스 통계, MariaDB 히스토그램은 `ANALYZE TABLE`로만 갱신된다.
- 문자셋 변환과 인덱스 변경을 같은 테이블에 한다면 ALTER 한 문장으로 합쳐 테이블을 한 번만 다시 만들고, 마지막에 `ANALYZE TABLE`을 돌린다.

## 핵심 코드
### 1) 조인 컬럼의 문자셋·콜레이션 확인
```sql
SELECT TABLE_NAME, COLUMN_NAME, CHARACTER_SET_NAME, COLLATION_NAME
  FROM information_schema.COLUMNS
 WHERE TABLE_SCHEMA = DATABASE()
   AND TABLE_NAME IN ('items', 'item_attachments', 'item_comments')
   AND COLUMN_NAME IN ('tenant_id', 'item_id');
```

### 2) DDL 없이 효과 미리 보기 (측정 전용)
```sql
-- 원래
EXPLAIN SELECT a.item_id, COUNT(f.file_seq) FROM items a
  LEFT JOIN item_attachments f ON f.tenant_id = a.tenant_id AND f.item_id = a.item_id
 WHERE a.tenant_id = 0 GROUP BY a.item_id ORDER BY a.created_date DESC LIMIT 20;
-- 변형: 넓은 쪽(A)을 좁은 쪽 문자셋으로 바꿔 F 인덱스를 쓰게 함
EXPLAIN SELECT a.item_id, COUNT(f.file_seq) FROM items a
  LEFT JOIN item_attachments f ON f.tenant_id = a.tenant_id
                              AND f.item_id = CONVERT(a.item_id USING utf8)
 WHERE a.tenant_id = 0 GROUP BY a.item_id ORDER BY a.created_date DESC LIMIT 20;
```

### 3) 사전 점검: 기준 콜레이션에 맞춘 ALTER 문 생성
```sql
SELECT t.TABLE_NAME, t.TABLE_COLLATION, t.ROW_FORMAT,
       CONCAT('ALTER TABLE `', t.TABLE_NAME, '` CONVERT TO CHARACTER SET utf8mb4 COLLATE ',
              (SELECT COLLATION_NAME FROM information_schema.COLUMNS
                WHERE TABLE_SCHEMA = DATABASE() AND TABLE_NAME = 'items' AND COLUMN_NAME = 'item_id'),
              IF(t.ROW_FORMAT IN ('Compact','Redundant'), ', ROW_FORMAT=DYNAMIC', ''), ';',
              ' ANALYZE TABLE `', t.TABLE_NAME, '`;') AS alter_sql
  FROM information_schema.TABLES t
 WHERE t.TABLE_SCHEMA = DATABASE()
   AND t.TABLE_NAME IN ('item_attachments', 'item_comments');
```

### 4) 점검을 통과했을 때만 실행 (파일 전체를 실수로 돌려도 안전하게)
```sql
SET @ok := (SELECT COUNT(*) = 0 FROM <점검 결과에서 문제 항목을 세는 쿼리>);
SET @sql := IF(@ok AND <대상 테이블이 아직 utf8인지>,
               'ALTER TABLE item_attachments CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci',
               'SELECT ''건너뜀'' AS result');
PREPARE stmt FROM @sql; EXECUTE stmt; DEALLOCATE PREPARE stmt;
```

### 5) 문자셋을 못 맞출 때 쿼리로 우회
```sql
-- 전: 승인자 목록(utf8)과 글(utf8mb4) 조인 → 게시판 조건이 승인자 쪽으로 안 넘어감
SELECT DISTINCT b.board_id
  FROM board_approvers a JOIN items b ON a.tenant_id = b.tenant_id AND a.board_id = b.board_id
 WHERE b.tenant_id = ? AND b.board_id = ?;
-- 후: 작은 쪽에서 먼저 찾고, 큰 쪽은 있는지만
SELECT DISTINCT a.board_id
  FROM board_approvers a
 WHERE a.tenant_id = ? AND a.board_id = ?
   AND EXISTS (SELECT 1 FROM items b WHERE b.tenant_id = a.tenant_id AND b.board_id = a.board_id);

-- 개수 쿼리: 이미 성립하는 조건을 상대 테이블에도 명시
SELECT COUNT(*) FROM items a JOIN boards p ON a.board_id = p.board_id
 WHERE a.tenant_id = ? AND a.board_id = ?
   AND p.board_id = ?;          -- 추가: p를 상수로 한 번만 읽게 함
```

## 주의할 점
- 앱과 같은 연결 콜레이션으로 측정한다. 클라이언트 기본 세션에서는 문제가 안 보일 수 있다.
- 변환 전후 결과가 같은지 반드시 확인한다. 같은 쪽 범위의 행 ID·표시 컬럼을 MD5로 묶어 비교하면 간단하다.
- 이미 변환된 DB에서는 "이미 같음"만 출력하고 아무것도 바꾸지 않게 만들어 두면 여러 번 실행해도 안전하다.
- `INNODB_TRX`·`PROCESSLIST`로 긴 트랜잭션을 보는 점검은 PROCESS 권한이 필요하다. 조회 전용 계정으로는 실패할 수 있다.
- 변환 후에도 느리면 다음 원인(행별 추가 조회, 페이지 전에 전체 정렬 등)을 따로 잰다. 문자셋은 조인 비용만 줄인다.

## 참고
- MySQL Reference Manual: Character Set and Collation Conversion, `ALTER TABLE ... CONVERT TO CHARACTER SET`
- MySQL Reference Manual: InnoDB Limits (index key prefix length), Persistent Optimizer Statistics
- MariaDB Knowledge Base: Engine-Independent Table Statistics
