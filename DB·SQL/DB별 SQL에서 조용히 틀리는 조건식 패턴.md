# DB별 SQL에서 조용히 틀리는 조건식 패턴

> 분류: DB·SQL · 최근 갱신: 2026-10-27

## 한 줄 요약
문법 오류 없이 실행되지만 결과가 0건이거나 엉뚱한 값이 나오는 SQL 패턴들 — 연산자 우선순위, `IN` 안의 `OR`, `BETWEEN` 인자 순서, 날짜에 숫자 빼기, 시각이 붙은 날짜 비교, DB마다 다른 함수, `ORDER BY` 없는 순서 의존.

## 내용
### Oracle: `||` 와 `-` 의 우선순위가 같다
- `'a' || ':' || sn - 1 || ':' || flag`는 `((('a'||':'||sn) - 1) || ...)`로 계산된다. 문자열에서 1을 빼게 되어 `ORA-01722: invalid number`가 난다.
- 산술은 반드시 괄호로 감싼다: `|| (sn - 1) ||`. `TO_CHAR(sn - 1)`로 명시적으로 문자열화해도 된다.

### MySQL: `IN ('01' OR '02' OR '03')`
- `IN (...)` 안이 값 목록이 아니라 OR 식 하나다. MySQL은 문자열을 숫자로 바꿔 `1 OR 16 OR 4` = `1`로 계산하므로 사실상 `IN (1)`이 된다.
- `'01'`은 숫자 1로 변환되어 맞지만 `'02'`, `'03'`은 빠진다. 일부만 조용히 누락되는 전형적인 경우다.
- 올바른 형태는 `IN ('01', '02', '03')`. Oracle은 같은 식에서 오류를 내므로 DB별 SQL을 복사할 때 특히 주의한다.

### `BETWEEN 큰값 AND 작은값`은 항상 거짓
- `BETWEEN a AND b`는 `>= a AND <= b`다. a가 b보다 크면 어떤 행도 맞지 않는다. 오류가 아니라 0건이므로 "버튼을 눌렀는데 아무 일도 안 일어남"으로 나타난다.
- 파라미터 이름이 `startTime`, `endTime`처럼 비슷하면 자리를 바꿔 쓰기 쉽다. 날짜 단위 비교는 `col >= 시작일 00:00 AND col < 다음날 00:00`로 쓰면 순서 실수와 시각 경계(23:59:59 이후 소수점 초) 문제가 함께 사라진다.

### 날짜 타입에 숫자를 빼면 DB마다 다르다
- MySQL: `CURDATE() - 1`은 날짜를 `20261015` 같은 정수로 바꿔 1을 뺀다. 매월 1일에는 `20261100`처럼 없는 날짜가 나온다. 날짜 연산은 `CURDATE() - INTERVAL 1 DAY`, `DATE_SUB(CURDATE(), INTERVAL 1 DAY)`를 쓴다.
- Oracle·Tibero: `SYSDATE - 1`은 정상적인 "하루 전"이다. 같은 식이 DB마다 의미가 다르므로 이식할 때 확인한다.
- 출력 형식도 명시한다. MySQL에서 DATE 컬럼을 그대로 SELECT하면 드라이버·버전에 따라 `2026-10-15`로 나올 수 있다. 외부 시스템이 `YYYYMMDD`를 기대하면 `DATE_FORMAT(col, '%Y%m%d')`, Oracle은 `TO_CHAR(col, 'YYYYMMDD')`로 고정한다. 여러 쿼리가 같은 파일로 내보낸다면 모든 쿼리에서 같은 형식인지 비교한다.

### 시각이 붙은 값끼리 `=` 비교
- Oracle `TO_DATE(created_date) = SYSDATE - 1`: 왼쪽은 자정(00:00:00), 오른쪽은 지금 시각이 붙은 하루 전이라 거의 절대 같지 않다. "매일 전날 데이터를 보내는 배치"가 사실상 아무것도 안 보내고 있어도 오류는 없다.
- 날짜 하루치를 고를 때는 범위로 비교한다: `col >= TRUNC(SYSDATE) - 1 AND col < TRUNC(SYSDATE)`. 컬럼을 가공하지 않아 인덱스도 탄다. `TO_CHAR(col, 'YYYY-MM-DD') = TO_CHAR(SYSDATE - 1, 'YYYY-MM-DD')`도 맞지만 인덱스를 못 쓴다.

### DB마다 없는 함수
- `now()`, `curdate()`, `IFNULL`, `DATE_FORMAT`은 MySQL 함수다. Oracle·Tibero에서는 `SYSDATE`, `TRUNC(SYSDATE)`, `NVL`, `TO_CHAR`를 쓴다. Oracle에서 `curdate()`를 호출하면 `ORA-00904: invalid identifier`가 난다.
- 호출하는 곳이 없는 쿼리 정의는 이런 오류가 숨어 있어도 드러나지 않는다. 정리할 때 호출처를 먼저 검색하고 지운다.

### `ORDER BY` 없는 결과 순서에 의존하는 코드
- 화면 목록을 "조회된 순서대로" 읽어 `"YNY"` 같은 문자열을 만들고, 서버는 다른 쿼리(`ORDER BY sn`)의 i번째 행에 i번째 글자를 넣는 식의 코드는 두 순서가 같다는 보장이 없다.
- MySQL InnoDB는 보통 기본키 순서로 나와 우연히 맞지만, Oracle은 실행계획에 따라 달라질 수 있다. 결과가 "엉뚱한 행에 값이 들어감"이라 데이터 오류로 번진다.
- 위치로 대응시키는 두 조회에는 같은 정렬 기준을 명시하고, 개수가 다를 수 있는 경우(한쪽은 JOIN, 한쪽은 단일 테이블)는 서버에서 길이를 확인한다.

## 핵심 코드
### 1) 조건식 바로 쓰기
```sql
-- Oracle 문자열 연결 안의 산술
SELECT 'x:' || (sn - 1) || ':' || flag FROM t;

-- MySQL 값 목록
WHERE category_code IN ('01', '02', '03')

-- 하루치 범위 비교 (Oracle)
WHERE created_date >= TRUNC(SYSDATE) - 1 AND created_date < TRUNC(SYSDATE)

-- 하루치 범위 비교 (MySQL, 파라미터 'YYYY-MM-DD')
WHERE sent_date >= STR_TO_DATE(#date#, '%Y-%m-%d')
  AND sent_date <  STR_TO_DATE(#date#, '%Y-%m-%d') + INTERVAL 1 DAY

-- 외부 전송용 날짜 형식 고정
DATE_FORMAT(IFNULL(due_date, CURDATE() - INTERVAL 1 DAY), '%Y%m%d')   -- MySQL
TO_CHAR(NVL(due_date, SYSDATE - 1), 'YYYYMMDD')                        -- Oracle
```

### 2) 하루 전 날짜: DB마다 다르게
```sql
-- MySQL/MariaDB
SELECT CURDATE() - INTERVAL 1 DAY;              -- 11월 1일 → 10월 31일 (CURDATE() - 1은 20261100)
SELECT DATE_SUB(CURDATE(), INTERVAL 1 DAY);
```
```sql
-- Oracle/Tibero
SELECT TRUNC(SYSDATE) - 1 FROM DUAL;            -- 어제 00:00:00
```

### 3) 기간 조건: BETWEEN 대신 반열림 구간 + 호출 전 순서 확인
```sql
-- iBATIS 2 매퍼 (MySQL·Oracle 공통 형태)
WHERE created_at >= #startDate#
  AND created_at <  #endDateExcl#     -- 종료일 다음 날 00:00
```
```java
if (start.isAfter(end)) {                       // 뒤바뀐 입력은 0건이 아니라 오류로
    throw new IllegalArgumentException("start > end");
}
Map<String, Object> param = new HashMap<>();
param.put("startDate", java.sql.Date.valueOf(start));               // LocalDate → DATE
param.put("endDateExcl", java.sql.Date.valueOf(end.plusDays(1)));
```

### 4) 위치로 대응시키는 두 조회는 정렬을 명시하고 개수를 확인
```sql
SELECT sn, flag FROM order_items WHERE order_id = #orderId# ORDER BY sn
```
```java
List<Map<String, Object>> rows = dao.list("selectItems", param);   // ORDER BY sn
if (flags.length() != rows.size()) {
    throw new IllegalStateException("count mismatch: " + flags.length() + " / " + rows.size());
}
for (int i = 0; i < rows.size(); i++) {
    rows.get(i).put("flag", String.valueOf(flags.charAt(i)));
}
```

### 5) DB별 매퍼 파일에 같은 쿼리가 모두 있는지 비교
```bash
diff <(grep -o 'id="[^"]*"' orders-oracle.xml | sort) \
     <(grep -o 'id="[^"]*"' orders-mysql.xml  | sort)
# 쿼리 본문 차이는 같은 id 블록끼리 diff로 본다
```

## 주의할 점
- 이 패턴들은 모두 "오류 없이 0건 또는 일부 누락"으로 나타난다. 외부 전송 배치라면 받는 쪽에서 "안 왔다"는 말이 나올 때까지 모른다. 배포 후 첫 실행 결과(행 수)를 꼭 확인한다.
- 같은 기능을 MySQL·Oracle·Tibero 세 파일로 유지할 때는 수정이 세 곳에 모두 들어갔는지, 반대로 한 DB에만 있던 오래된 수정이 다른 DB에 빠져 있지 않은지 diff로 비교한다.

## 참고
- Oracle 문서: Operator Precedence, TRUNC(date)
- MySQL 문서: Date and Time Arithmetic, DATE_FORMAT, IN operator type conversion
