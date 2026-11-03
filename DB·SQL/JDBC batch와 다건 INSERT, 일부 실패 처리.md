# JDBC batch와 다건 INSERT, 일부 실패 처리

> 분류: DB·SQL · 최근 갱신: 2026-11-02

## 한 줄 요약
여러 행을 넣을 때 batch(`addBatch`/`executeBatch`)는 네트워크 왕복을 줄이지만 효과와 실패 동작이 드라이버마다 다르고, 다건 VALUES는 문장 전체가 실패한다. "한 행이 실패해도 나머지는 넣기"가 필요하면 한 건씩 실행 + 마지막 한 번 커밋, 또는 다건 실패 시 반으로 나눠 재시도한다.

## 내용
### 세 가지 방식
| 방식 | 왕복 | 한 행이 실패하면 |
|---|---|---|
| 한 건씩 `executeUpdate` | N번 | 그 행만 예외. try/catch로 건너뛰면 나머지는 모두 들어감 |
| batch `addBatch` → `executeBatch` | 드라이버에 따라 1~N번 | 드라이버마다 다름. MySQL·MariaDB 계열은 대개 계속 실행하고 실패 건만 표시, Oracle은 실패 지점에서 멈춰 뒤쪽은 실행 안 됨(`BatchUpdateException.getUpdateCounts()`로 확인) |
| 다건 INSERT 한 문장 `VALUES (...), (...)` | 묶음당 1번 | 문장 전체가 실패(묶음 모두 안 들어감) |

### batch가 실제로 묶어 보내는지
- Oracle·Tibero의 일반 INSERT: 드라이버가 묶어 보낸다.
- MySQL Connector/J: 접속 URL에 `rewriteBatchedStatements=true`가 없으면 사실상 한 건씩 보낸다.
- MariaDB Connector/J: 일반 INSERT는 묶어 보낸다(버전에 따라 bulk/pipelining).
- **프로시저 호출(`CallableStatement`)**: 대부분 드라이버가 안에서 한 건씩 실행한다. batch의 이득은 문장 준비를 한 번 하는 정도.
- iBATIS(`startBatch` → `insert` 반복 → `executeBatch`)와 MyBatis `ExecutorType.BATCH`도 안에서는 JDBC batch다.

### 커밋은 한 번으로
- 자동 커밋을 끄고 한 건씩 실행하다 한 문장이 실패해도, MySQL/MariaDB·Oracle·Tibero 모두 **실패한 문장만 롤백**하고 앞서 성공한 행은 트랜잭션에 남는다. 그래서 실패 건만 건너뛰고 마지막에 한 번 커밋할 수 있다.
- 이미 트랜잭션 안(같은 연결)에서 INSERT를 반복하고 있다면 남는 비용은 같은 망 DB까지의 왕복뿐(보통 1건 1ms 안팎)이라, 측정으로 느리다는 근거가 나오기 전에는 batch로 바꿀 이유가 약하다.

### 다건 INSERT에서 일부 실패 견디기
- 묶음(예: 100건) 문장이 실패하면 그 묶음을 반으로 나눠 다시 시도한다. 나쁜 행 1개를 찾는 데 문장 10여 개면 된다. 한 건까지 내려가서도 실패하면 그 행만 건너뛴다.
- 연결 오류(SQLState `08xxx`)는 나눠도 소용없으므로 바로 중단한다.
- 미리 거를 수 있는 원인(컬럼 길이 초과, 빈 필수값, 4바이트 문자와 문자셋)을 넣기 전에 정리하면 재시도가 거의 생기지 않는다.

### DB별 다건 INSERT 모양
- MySQL/MariaDB: `INSERT INTO t (...) VALUES (...), (...)`.
- Oracle: `INSERT ALL INTO t ... SELECT 1 FROM DUAL`에서 시퀀스 `NEXTVAL`은 **모든 행에 같은 값**이 들어가 PK가 충돌한다. 값들을 `UNION ALL`로 묶은 뒤 바깥 SELECT에서 시퀀스를 붙인다.
- 묶음 크기는 문장 길이·바인드 변수 개수 한도(드라이버·DB별)를 넘지 않게 정한다.

### 프로시저가 있는 테이블
- 다른 팀이 소유한 DB에 프로시저로 넣고 있다면, 테이블에 직접 다건 INSERT하는 대신 그 팀에 다건 프로시저를 요청하는 편이 유지보수에 낫다. 프로시저 안에서 하는 부가 처리(카운터 갱신, 큐 등록)를 몰라도 되고, 테이블 구조가 바뀌어도 인터페이스만 같으면 된다.
- 내용이 같은 대상끼리 묶어 "공통 내용 + ID 목록(구분자 문자열)"으로 넘기면 인자가 단순하다. 문자열 한도(Oracle SQL 안 VARCHAR2 4000바이트)를 기준으로 "N명 또는 4000바이트 중 먼저 닿는 쪽"에서 끊고, ID에 구분자가 들어 있으면 미리 거른다.

## 핵심 코드
### 1) 한 건씩 실행, 실패만 건너뛰고 커밋 한 번
```java
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement("INSERT INTO notifications (user_id, title, link) VALUES (?, ?, ?)")) {
    conn.setAutoCommit(false);
    int inserted = 0;
    for (Target t : targets) {
        try {
            ps.setString(1, t.userId); ps.setString(2, t.title); ps.setString(3, t.link);
            ps.executeUpdate();
            inserted++;
        } catch (SQLException e) {
            log.warn("skip userId={}, sqlState={}", t.userId, e.getSQLState());   // 실패 문장만 롤백됨
        }
    }
    conn.commit();
}
```

### 2) batch
```java
for (Target t : targets) {
    ps.setString(1, t.userId); ps.setString(2, t.title); ps.setString(3, t.link);
    ps.addBatch();                     // 메모리에 쌓기만
}
int[] counts = ps.executeBatch();      // 한 번에 보냄, 결과는 건별 배열
// MySQL은 URL에 rewriteBatchedStatements=true 가 있어야 실제로 묶임
```

### 3) 다건 실패 시 반으로 나눠 재시도
```java
int insertWithSplit(Connection conn, List<Target> rows) throws SQLException {
    try {
        return insertMulti(conn, rows);                       // VALUES 여러 줄, 문장 1개
    } catch (SQLException e) {
        if (e.getSQLState() != null && e.getSQLState().startsWith("08")) throw e;   // 연결 오류
        if (rows.size() == 1) { log.warn("skip {}", rows.get(0).userId); return 0; }
        int mid = rows.size() / 2;
        return insertWithSplit(conn, rows.subList(0, mid)) + insertWithSplit(conn, rows.subList(mid, rows.size()));
    }
}
```

### 4) Oracle 다건 INSERT와 시퀀스
```sql
INSERT INTO notifications (id, user_id, title, created_at)
SELECT notifications_seq.NEXTVAL, x.user_id, x.title, SYSDATE
FROM (
    SELECT ? AS user_id, ? AS title FROM DUAL
    UNION ALL SELECT ?, ? FROM DUAL
    UNION ALL SELECT ?, ? FROM DUAL
) x;
```

## 주의할 점
- 성능 개선의 대부분이 batch가 아니라 다른 데서 나오는 경우가 많다. 예를 들어 행마다 중간 서버를 HTTP로 거치며 조회 여러 번을 하던 구조라면, 그 중간 단계를 없애는 것만으로 1건당 작업이 크게 줄고, INSERT를 묶는 것은 그다음 문제다.
- 날짜는 `NOW()`/`SYSDATE` 대신 바인드 값으로 넘기면 DB별 SQL 차이를 줄일 수 있다.

## 참고
- JDBC API: Statement.addBatch, executeBatch, BatchUpdateException
- MySQL Connector/J: rewriteBatchedStatements
- Oracle 문서: INSERT ALL과 시퀀스
