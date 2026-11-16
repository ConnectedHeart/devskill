# WITH RECURSIVE로 상위 계층 따라 올라가기

> 분류: DB·SQL · 최근 갱신: 2026-11-12

## 한 줄 요약
"조건이 맞는 동안 부모로 계속 올라간다" 같은 while 반복은 재귀 CTE(`WITH RECURSIVE`)로 SQL 한 번에 옮길 수 있고, 같은 방식으로 루트까지의 전체 경로 문자열도 만든다. 순환 데이터 대비 단계 제한과 "멈춘 행 고르기"를 꼭 넣는다.

## 내용
### 재귀 CTE가 도는 방식
`WITH RECURSIVE 이름 AS (시작 SELECT UNION ALL 반복 SELECT)`
1. 시작 SELECT를 한 번 실행해 첫 행들을 만든다.
2. 반복 SELECT가 바로 앞 단계에서 만든 행만 읽어 다음 행을 만든다.
3. 반복 SELECT가 행을 더 만들지 않으면 멈춘다.
4. 모든 단계의 행을 모은 것이 결과다.

MySQL 8, MariaDB 10.2 이상, PostgreSQL, Oracle 11gR2 이상(`WITH` 재귀, `RECURSIVE` 키워드 없음)에서 쓸 수 있다.

### 예 1: 조건이 맞는 동안 위로 올라가 "멈춘 부서" 찾기
부서마다 "상위 부서의 처리를 따른다(Y)" 플래그가 있고, 실제 처리 부서는 위로 올라가다 처음으로 Y가 아닌 부서라고 하자.
- 시작 행: 모든 부서에서 하나씩 출발. "원래 부서"(끝까지 고정)와 "지금 보고 있는 부서"(올라갈수록 바뀜), 단계 `lvl=0`을 담는다.
- 반복 행: 지금 부서가 Y이면 부모 부서로 한 칸 올라간 행을 만든다.
- 결과에서 원래 부서마다 **더 올라가지 않은 행**(지금 부서가 Y가 아니거나 더 갈 부모가 없음)을 고른다. 가장 큰 `lvl` 행을 고르는 것과 같다.

### 예 2: 루트 바로 아래까지 이름을 이어 전체 경로 만들기
- 시작 행: 부서 자신의 이름을 경로로 둔다.
- 반복 행: 부모가 루트(회사 노드 등)가 아니면 `CONCAT(부모이름, '/', 경로)`로 앞에 붙이며 올라간다.
- 원래 부서마다 가장 많이 올라간 행의 경로가 전체 경로다.
- 이미 저장된 "경로 컬럼"이 있어도 실제 부모 관계와 어긋난 행이 있을 수 있다. 부모 코드로 직접 거슬러 올라가는 편이 정확하다.

### 데이터 이상 대비
- **순환**(A의 부모가 B, B의 부모가 A): 반복이 끝나지 않는다. `lvl < 20`처럼 단계 상한을 둔다. MySQL은 `cte_max_recursion_depth`(기본 1000)에서 오류로 멈춘다.
- **자기 자신을 부모로 가리키는 행**: 반복 조건에 `parent_id <> id`를 넣는다.
- **부모가 없는(지워진) 행**: 조인이 실패해 그 단계에서 멈춘다. 결과에서 빠지지 않도록 "더 올라가지 않은 행"을 기준으로 고른다.

### 가정 시나리오 시험
부서를 옮기거나 플래그를 바꾼 경우를 보고 싶으면 트랜잭션 안에서 UPDATE → 쿼리 실행 → `ROLLBACK`한다. 커밋 전 값은 다른 연결에 보이지 않아 운영 중인 화면에 영향이 없다.

## 핵심 코드
### 1) 처리 부서 찾기 (MySQL 8 / MariaDB)
```sql
WITH RECURSIVE box AS (
  SELECT d.id AS dept_id, d.name AS dept_name,
         d.id AS cur_id, IFNULL(d.follow_parent,'N') AS cur_yn,
         d.parent_id AS cur_parent, 0 AS lvl
    FROM departments d
   WHERE d.use_yn = 'Y'
  UNION ALL
  SELECT b.dept_id, b.dept_name,
         p.id, IFNULL(p.follow_parent,'N'), p.parent_id, b.lvl + 1
    FROM box b
    JOIN departments p ON p.id = b.cur_parent
   WHERE b.cur_yn = 'Y'              -- Y인 동안만 위로
     AND b.cur_parent <> b.cur_id    -- 자기 자신을 가리키면 멈춤
     AND b.lvl < 20                  -- 순환 데이터 대비
)
SELECT b.dept_id, b.dept_name, b.cur_id AS box_dept_id, b.lvl
  FROM box b
 WHERE b.lvl = (SELECT MAX(x.lvl) FROM box x WHERE x.dept_id = b.dept_id);
```

### 2) 전체 경로 만들기 (루트 노드는 경로에서 뺌)
```sql
WITH RECURSIVE pth AS (
  SELECT d.id AS dept_id, d.parent_id AS cur_parent,
         CAST(d.name AS CHAR(1000)) AS path, 0 AS lvl   -- 길이를 넉넉히 잡아 둔다
    FROM departments d
  UNION ALL
  SELECT t.dept_id, p.parent_id, CONCAT(p.name, '/', t.path), t.lvl + 1
    FROM pth t
    JOIN departments p ON p.id = t.cur_parent
   WHERE p.parent_id <> 'ROOT'        -- 회사 노드(부모가 ROOT)는 넣지 않음
     AND t.lvl < 20
)
SELECT dept_id, path
  FROM pth t
 WHERE t.lvl = (SELECT MAX(x.lvl) FROM pth x WHERE x.dept_id = t.dept_id);
```

### 3) 가정 시나리오를 롤백으로 시험
```sql
START TRANSACTION;
UPDATE departments SET parent_id = 'team1', follow_parent = 'Y' WHERE id = 'team2';
-- 위 SELECT 실행해서 결과 확인
ROLLBACK;
```

## 주의할 점
- 시작 행의 문자열 컬럼 길이가 결과 컬럼 길이를 정한다. 경로를 이어 붙일 때는 `CAST(... AS CHAR(1000))`처럼 넉넉히 잡지 않으면 잘리거나 오류가 난다.
- 이름을 구분자로 이을 때 이름 안에 그 구분자(공백, `/`)가 들어 있으면 경계가 헷갈린다. 먼저 `WHERE name LIKE '%/%'`로 확인하거나 탭을 쓴다.
- 루트를 경로에서 빼면 다른 루트(다른 회사) 아래의 같은 이름 부서가 같은 경로로 나온다. 필요하면 루트 이름 컬럼을 따로 붙인다.
- 애플리케이션 코드의 판정 반복과 SQL이 같은 기준인지(어디서 멈추는지, 빈 플래그를 N으로 보는지) 맞춘 뒤 대표 사례로 대조한다.
- 접속 문자셋·콜레이션이 서버와 다르면 문자열 비교에서 콜레이션 오류가 날 수 있다. 직접 실행할 때는 앱과 같은 `SET NAMES ... COLLATE ...`를 먼저 둔다.

## 참고
- MySQL Reference Manual — WITH (Common Table Expressions), Recursive CTEs
- MariaDB Knowledge Base — Recursive Common Table Expressions
