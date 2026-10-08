# 시각 컬럼 타입 선택(epoch millis와 날짜 타입)

> 분류: DB·SQL · 최근 갱신: 2026-10-07

## 한 줄 요약
애플리케이션이 현재 시각과 직접 비교하는 만료 시각은 epoch millis 숫자로 두는 것이 타임존 문제에서 안전하고, 사람이 읽기만 하는 기록용 시각은 날짜 타입이 편하다.

## 내용
- **숫자(epoch millis)의 장점**: `System.currentTimeMillis()`끼리 숫자 비교라 타임존이 끼어들 여지가 없다. MySQL/Oracle 등 DB 종류가 달라도 같은 SQL을 쓴다.
- **날짜 타입으로 바꾸면 생기는 것**
  - `DATETIME`/`DATE`에는 타임존 정보가 없어 JDBC 드라이버가 JVM 타임존(또는 커넥션 설정) 기준으로 변환한다. 서버 간 타임존이나 드라이버 설정이 다르면 같은 행을 몇 시간 어긋나게 읽는다.
  - 수명이 분 단위인 토큰처럼 여유가 짧은 만료 판정에서는 어긋남이 바로 장애가 된다(만료된 값을 계속 쓰거나, 요청마다 재발급).
  - 날짜 타입이 되면 `NOW()`/`SYSDATE`로 비교하고 싶어지는데, 그러면 애플리케이션 서버 시계와 DB 시계·타임존 차이까지 끼어든다. 비교 기준은 한쪽(애플리케이션)에서만 넘긴다.
  - 서머타임이 있는 지역에서 로컬 시각으로 저장하면 전환 시간대에 1시간이 겹친다.
- **절충**: 로직이 읽는 만료 시각은 숫자로 두고, 로직에서 쓰지 않는 기록용 컬럼(마지막 갱신 시각 등)만 날짜 타입으로 두되 DB가 UTC 현재 시각을 직접 채우게 한다. 조회 시 로컬 시각과 시차가 있다는 점만 기억하면 된다.
- **숫자 컬럼을 사람이 읽을 때**: 조회에서 변환한다.

## 예시
```sql
-- epoch millis를 읽기 좋게 (MySQL/MariaDB)
SELECT FROM_UNIXTIME(expire_time / 1000) AS expire_dt FROM app_token;

-- DB가 UTC 현재 시각을 채우기
-- MySQL/MariaDB
UPDATE app_token SET update_time = utc_timestamp() WHERE token_key = ?;
-- Oracle
UPDATE app_token SET update_time = SYS_EXTRACT_UTC(SYSTIMESTAMP) WHERE token_key = ?;

-- 숫자 컬럼을 날짜 타입으로 변경: 두 DB 모두 먼저 비운다
-- MySQL/MariaDB
DELETE FROM app_token;
ALTER TABLE app_token MODIFY update_time DATETIME NOT NULL COMMENT '마지막 갱신 시각(UTC)';
-- Oracle
DELETE FROM app_token;
COMMIT;
ALTER TABLE app_token MODIFY (update_time DATE);
```

## 주의할 점
- Oracle은 값이 들어 있는 컬럼의 타입을 NUMBER에서 DATE로 바꿀 수 없다(`ORA-01439`). 비우거나 새 컬럼을 만들어 옮긴다.
- MySQL은 행이 남아 있으면 숫자를 날짜로 변환하지 못해 strict 모드에서는 `ALTER`가 실패하고, 아니면 `0000-00-00`이 들어간다.
- MySQL `MODIFY`는 컬럼 정의 전체를 다시 쓰는 것이라 `NOT NULL`·`COMMENT`를 빼면 사라진다. Oracle `MODIFY`는 적지 않은 제약이 유지된다.
- 실행 중인 서버가 있으면 컬럼 변경과 새 쿼리 배포를 같은 시점에 한다. 컬럼만 먼저 바꾸면 기존 코드가 숫자를 넣으려다 실패한다.
- MySQL `DATETIME`과 Oracle `DATE`는 기본이 초 단위다.

## 참고
- MySQL Reference Manual — Date and Time Functions, ALTER TABLE
- Oracle Database SQL Language Reference — ALTER TABLE
