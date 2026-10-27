# AUTO_INCREMENT가 건너뛰며 증가하는 이유

> 분류: DB·SQL · 최근 갱신: 2026-10-27

## 한 줄 요약
MySQL/MariaDB에서 ID가 3씩(또는 N씩) 증가하면 대개 `auto_increment_increment` 설정 때문이고, Galera 클러스터가 노드 간 ID 충돌을 막으려고 자동으로 넣는 값이다.

## 내용
### 증가 폭을 정하는 변수
- `auto_increment_increment`: 증가 폭
- `auto_increment_offset`: 시작 위치
- INSERT 문이 ID 컬럼을 넣지 않으면 증가 폭은 코드가 아니라 이 설정이 결정한다. 값이 3이면 **모든** `AUTO_INCREMENT` 컬럼이 3씩 증가한다.

### Galera 클러스터
- `wsrep_auto_increment_control=ON`이면 노드 수에 맞춰 `auto_increment_increment`를 자동 설정하고 노드마다 `auto_increment_offset`을 다르게 준다(3노드면 1, 2, 3).
- 여러 노드에 동시에 쓰더라도 ID가 겹치지 않게 하려는 것이다. 클러스터에서 이 값을 1로 바꾸면 노드 간 ID 충돌이 날 수 있다.

### 설정 때문인지 중복 요청 때문인지 구분
| 관찰 | 원인 |
|---|---|
| 등록 1건당 행이 1개이고 ID만 N씩 뛴다 | DB 설정 (정상) |
| 같은 내용의 행이 N개씩 쌓인다 | 요청이 N번 들어옴 → 화면·호출 쪽 중복을 확인 |

### 방금 넣은 ID 가져오기
- `SELECT MAX(id)`로 가져오면 동시에 다른 연결이 넣은 행의 ID를 받을 수 있다. 이후 그 ID로 링크나 후속 처리를 하면 엉뚱한 행을 가리킨다.
- `LAST_INSERT_ID()`는 연결(세션)별 값이라 동시 등록에도 안전하다.

## 핵심 코드
### 1) 설정 확인과 방금 넣은 ID (MySQL/MariaDB)
```sql
SHOW VARIABLES WHERE Variable_name IN
  ('auto_increment_increment', 'auto_increment_offset',
   'wsrep_auto_increment_control', 'wsrep_cluster_size');

INSERT INTO notice (title) VALUES ('...');
SELECT LAST_INSERT_ID();
```
`wsrep_cluster_size`는 상태 변수라 `SHOW STATUS LIKE 'wsrep_cluster_size';`로 본다.

### 2) 설정 때문인지 중복 요청 때문인지 (같은 분에 같은 내용이 여러 행인지)
```sql
SELECT title,
       DATE_FORMAT(created_at, '%Y-%m-%d %H:%i') AS min_slot,
       COUNT(*) AS cnt, MIN(id) AS first_id, MAX(id) AS last_id
  FROM notice
 WHERE created_at >= NOW() - INTERVAL 1 DAY
 GROUP BY title, DATE_FORMAT(created_at, '%Y-%m-%d %H:%i')
HAVING COUNT(*) > 1;            -- 결과가 없으면 중복 요청이 아니라 증가 폭 설정
```

### 3) JDBC로 생성 키 받기 (MAX(id) 금지)
```java
long id = -1;
try (PreparedStatement ps = conn.prepareStatement(
        "INSERT INTO notice (title) VALUES (?)", Statement.RETURN_GENERATED_KEYS)) {
    ps.setString(1, title);
    ps.executeUpdate();
    try (ResultSet rs = ps.getGeneratedKeys()) {
        if (rs.next()) {
            id = rs.getLong(1);          // 이 연결이 넣은 행의 ID
        }
    }
}
```

### 4) iBATIS 2 매퍼에서 키 받기: MySQL과 Oracle
```xml
<!-- MySQL/MariaDB: INSERT 뒤에 selectKey -->
<insert id="insertNotice" parameterClass="java.util.Map">
    INSERT INTO notice (title) VALUES (#title#)
    <selectKey resultClass="long" keyProperty="id">
        SELECT LAST_INSERT_ID()
    </selectKey>
</insert>

<!-- Oracle: INSERT 앞에서 시퀀스로 먼저 받는다 -->
<insert id="insertNotice" parameterClass="java.util.Map">
    <selectKey resultClass="long" keyProperty="id">
        SELECT notice_seq.NEXTVAL FROM DUAL
    </selectKey>
    INSERT INTO notice (id, title) VALUES (#id#, #title#)
</insert>
```
`insert()`의 반환값과 파라미터 Map의 `id`에 키가 들어온다.

## 주의할 점
- ID가 연속이라는 가정으로 코드를 짜지 않는다. `id > :lastSeen`, `MAX(id)` 같은 크기 비교는 건너뛰어도 문제없지만, "다음 ID = 현재 + 1" 같은 계산은 깨진다.
- 롤백된 INSERT, `INSERT ... ON DUPLICATE KEY`, 서버 재시작 등으로도 번호가 비므로 빈 번호 자체는 이상 신호가 아니다.
- DBMS마다 방금 넣은 키를 얻는 방법이 다르다(Oracle은 시퀀스, `RETURNING`). 매퍼를 DB별로 따로 확인한다.

## 참고
- MariaDB 문서: Galera Cluster System Variables (`wsrep_auto_increment_control`)
- MySQL 문서: `auto_increment_increment`, `LAST_INSERT_ID()`
