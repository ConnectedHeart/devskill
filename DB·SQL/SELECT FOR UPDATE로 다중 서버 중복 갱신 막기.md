# SELECT FOR UPDATE로 다중 서버 중복 갱신 막기

> 분류: DB·SQL · 최근 갱신: 2026-11-16

## 한 줄 요약
서버가 여러 대일 때 "한 번만 일어나야 하는 갱신"은 JVM 잠금으로 막을 수 없으므로, 대상 행을 `SELECT ... FOR UPDATE`로 잠그고 잠근 뒤 최신 값을 다시 판정한다.

## 내용
- **문제 상황**: 한 번 쓰면 폐기되는 값(로테이션되는 refresh token 등)을 DB에 두고, 여러 요청이 동시에 "만료됐으니 갱신"을 시도하는 경우. 먼저 간 요청만 성공하고 늦은 요청은 실패하며, 실패 처리(행 삭제 등)가 정상 데이터를 망가뜨린다.
- **JVM 잠금의 한계**: `synchronized`나 `ReentrantLock`은 서버 한 대 안에서만 유효하다. 로드밸런서 뒤에서 두 요청이 다른 서버로 가면 서로를 모른다.
- **행 잠금 방식**
  1. 평소 요청은 잠금 없이 읽어서, 아직 유효하면 그대로 쓴다.
  2. 갱신이 필요하면 트랜잭션을 열고 그 행을 `SELECT ... FOR UPDATE`로 읽는다. 다른 서버가 같은 행을 갱신 중이면 그쪽이 커밋할 때까지 DB에서 기다린다.
  3. 잠금을 얻은 뒤 방금 읽은 최신 행을 **다시 판정**한다. 기다리는 동안 다른 쪽이 이미 갱신했으면 외부 호출 없이 그 값을 쓴다.
  4. 아니면 갱신하고 저장한 뒤 커밋한다. 커밋과 함께 잠금이 풀린다.
- **트랜잭션이 필수**: 자동 커밋 상태에서는 잠금이 SELECT 직후 풀려 아무것도 막지 못한다. 잠금 조회와 저장이 한 트랜잭션에 있어야 한다.
- **JVM 잠금과 함께 쓰면 좋은 이유**: 행 잠금만 쓰면 기다리는 요청마다 DB 커넥션을 하나씩 잡고 대기한다. 같은 서버의 요청은 메모리 잠금으로 먼저 줄을 세우고, 서버 간 중복만 행 잠금이 막게 하면 커넥션을 아낀다.

| | JVM 잠금 | DB 행 잠금 |
|---|---|---|
| 범위 | 같은 서버 안 | 모든 서버 |
| 기다리는 동안 | 커넥션을 잡지 않음 | 요청마다 커넥션 하나를 잡고 대기 |
| 없으면 | 결과는 맞지만 커넥션을 더 씀 | 서버 간 중복 갱신 발생 |

### 대표 행 하나를 잠가 "범위 재계산"을 하나씩 돌리기
- **문제 상황**: 부서 트리처럼 여러 행에서 파생값(예: 부서마다 "처리 부서명 + 팀명")을 계산해 저장하는 작업이 저장·이동마다 돈다. 두 요청이 동시에 돌면 먼저 읽은 오래된 트리로 계산한 쪽이 나중에 저장해 최신 값을 덮어쓸 수 있다.
- **해결**: 재계산을 시작할 때 그 범위를 대표하는 행 하나(테넌트·회사의 루트 행 등)를 `SELECT ... FOR UPDATE`로 먼저 잠근다. 같은 범위의 재계산은 서버가 여러 대여도 한 번에 하나씩만 돈다. 잠금은 트랜잭션이 끝날 때까지 유지되므로 "잠금 → 목록 읽기 → 계산 → 바뀐 행만 UPDATE"를 한 트랜잭션에 둔다.
- 데이터 변경(저장·이동)을 먼저 커밋하고 그 뒤에 재계산을 부르면, 마지막에 도는 재계산이 항상 최신 데이터를 읽는다.
- 시험: 다른 DB 세션이 대표 행을 10초 잡고 있게 한 뒤 재계산을 부르면 약 10초 기다렸다가 정상으로 끝나는지 본다.
- **잠근 뒤에 읽어야 하는 이유(REPEATABLE READ)**: MySQL/InnoDB 기본 격리 수준은 트랜잭션의 첫 일반 조회 시점 스냅샷을 계속 본다. 잠금보다 먼저 목록을 읽어 두면, 잠금을 기다리는 사이 앞선 재계산이 커밋한 값을 못 보고 옛 스냅샷으로 계산한다.
- **대가**: 대표 행을 트랜잭션 끝까지 쥐고 있으므로 재계산이 느리거나 다른 긴 트랜잭션에 막히면, 같은 행을 잠그려는 요청이 줄줄이 기다린다. 막으려는 경쟁이 "몇십 ms 안에 겹치는 드문 경우이고 다시 실행하면 바로잡히는" 정도라면 잠금을 빼는 것도 선택지다.
- **잠금 없는 대안: 조건부 UPDATE**: 읽은 값을 조건에 넣어 `UPDATE ... SET value = ? WHERE id = ? AND value = <읽을 때 값>`으로 쓴다. 읽은 뒤 누가 바꿨으면 0건이 되어 덮어쓰지 않는다(낙관적 방식). 긴 잠금이 없어지고 덮어쓰기 경쟁도 대부분 막힌다.
- 운영에서 "버튼을 누르니 다른 기능이 멈췄다"는 문의가 오면, 로컬에서 한 행만 잠겨도 운영은 PK가 없거나 트리거가 큰 트랜잭션을 만들어 넓게 잠길 수 있다. 멈춘 동안 열린 트랜잭션과 잠금 대기를 먼저 조회한다(아래 5).

## 핵심 코드
### 1) 잠금 조회와 갱신 SQL
```sql
-- 트랜잭션 안에서
SELECT token_key, access_token, refresh_token, access_expire_time
  FROM user_token
 WHERE token_key = ?
   FOR UPDATE;

-- (필요할 때만) 외부 호출로 갱신한 뒤
UPDATE user_token
   SET access_token = ?, refresh_token = ?, access_expire_time = ?
 WHERE token_key = ?;
COMMIT;
```
MySQL/MariaDB, Oracle, Tibero에서 구문이 같다.

### 2) 잠금 대기 시간 상한
```sql
-- MySQL/MariaDB: 이 세션의 행 잠금 대기 상한(초)
SET SESSION innodb_lock_wait_timeout = 10;

-- Oracle: 문장 단위로 대기 상한(초)
SELECT access_token, refresh_token, access_expire_time
  FROM user_token
 WHERE token_key = ?
   FOR UPDATE WAIT 10;
```

### 3) JDBC: 자동 커밋 끄기 → 잠금 → 다시 판정 → 저장 → 커밋
```java
public Token refreshIfNeeded(Connection conn, String key) throws SQLException, IOException {
    conn.setAutoCommit(false);                       // 끄지 않으면 SELECT 직후 잠금이 풀린다
    try {
        Token t = selectForUpdate(conn, key);        // 다른 서버가 갱신 중이면 여기서 대기
        long now = System.currentTimeMillis();
        if (t.getAccessExpireTime() <= now + 60_000L) {   // 잠근 뒤 최신 값으로 다시 판정
            t = callRefresh(t.getRefreshToken());    // 외부 호출(짧은 타임아웃)
            update(conn, key, t);
        }
        conn.commit();                               // 커밋과 함께 잠금 해제
        return t;
    } catch (SQLException | IOException | RuntimeException e) {
        conn.rollback();
        throw e;
    } finally {
        conn.setAutoCommit(true);
    }
}
```
Spring에서는 이 메서드를 `@Transactional(propagation = Propagation.REQUIRES_NEW)` 별도 빈으로 두고, 바깥에서 `synchronized`로 같은 서버 요청을 먼저 줄 세운다.

### 4) 대표 행 잠금으로 범위 재계산 직렬화
```sql
-- 범위(테넌트)의 루트 행 하나만 잠근다
SELECT id FROM departments
 WHERE tenant_id = ? AND parent_id = 'ROOT'
   FOR UPDATE;
```
```java
@Transactional   // 잠금은 이 메서드가 끝나 커밋될 때까지 유지
public int refreshDerivedNames(int tenantId) {
    deptDao.lockRoot(tenantId);                        // 같은 테넌트 재계산은 여기서 줄을 선다
    List<Dept> all = deptDao.findAll(tenantId);        // 잠근 뒤에 읽어야 최신 트리
    int changed = 0;
    for (Dept d : all) {
        String value = computeName(d, all);
        if (!value.equals(d.getDerivedName())) {       // 바뀐 행만 UPDATE
            deptDao.updateDerivedName(d.getId(), value);
            changed++;
        }
    }
    return changed;
}
```

### 5) 멈췄을 때 열린 트랜잭션·잠금 대기 보기 (MySQL 8 / MariaDB)
```sql
-- 오래 열린 트랜잭션
SELECT trx_id, trx_state, trx_started,
       TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS sec, trx_mysql_thread_id, trx_query
  FROM information_schema.innodb_trx
 ORDER BY trx_started;
-- MariaDB: 누가 누구를 막는지
SELECT * FROM information_schema.innodb_lock_waits;
-- 막고 있는 연결을 끊으면 그 트랜잭션은 롤백된다
-- KILL <trx_mysql_thread_id>;
```

### 6) 잠금 없는 조건부 UPDATE
```sql
-- 읽을 때 값이 그대로일 때만 덮어쓴다 (영향 행 0이면 누가 먼저 바꾼 것)
UPDATE departments
   SET derived_name = ?
 WHERE id = ? AND tenant_id = ? AND derived_name = ?;
```

## 주의할 점
- 잠금을 쥔 동안 외부 호출을 하면 그 시간만큼 커넥션과 행 잠금이 유지된다. 외부 호출 타임아웃을 짧게 둔다.
- 잠금 조회는 기본 키 등 인덱스로 한 행만 집어야 한다. 인덱스를 못 타면 넓은 범위가 잠길 수 있다.
- 무중단 배포 중 행 잠금을 쓰지 않는 이전 버전과 섞여 도는 동안에는 중복이 생길 수 있다.
- 이 구간의 트랜잭션은 호출한 쪽 트랜잭션과 분리해야 잠금이 바로 풀린다(→ `Spring 트랜잭션 전파와 자동 커밋`).

## 참고
- MySQL Reference Manual — Locking Reads
- Oracle SQL Language Reference — SELECT, for_update_clause
