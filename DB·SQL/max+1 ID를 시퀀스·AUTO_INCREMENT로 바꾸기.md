# max+1 ID를 시퀀스·AUTO_INCREMENT로 바꾸기

> 분류: DB·SQL · 최근 갱신: 2026-11-17

## 한 줄 요약
`SELECT MAX(id)+1`로 PK를 정하면 서버 여러 대·서로 다른 서비스 객체에서 동시에 저장할 때 같은 번호가 나와 PK 중복으로 실패한다. DB가 번호를 주게(Oracle 시퀀스, MySQL AUTO_INCREMENT) 바꾸고 INSERT에서 키를 돌려받으면 원천적으로 사라지지만, 운영 DB 반영 SQL과 배포 순서를 함께 설계해야 한다.

## 내용
### max+1이 겹치는 이유
- 두 저장이 거의 동시에 `MAX+1`을 읽으면 같은 값을 받는다. `synchronized`는 한 JVM 안의 한 객체에서만 막으므로, 서버가 여러 대이거나 웹·모바일처럼 다른 빈이 같은 테이블에 쓰면 무의미하다.
- 번호를 메서드 첫머리에서 읽고 검사·삭제를 한 뒤 INSERT하면 겹칠 수 있는 시간이 더 길어진다.

### 해결 방법 비교
| 방법 | 장점 | 단점 |
|---|---|---|
| 겹치면 새 트랜잭션으로 재시도 | 스키마 변경 없음 | 우회 코드가 남음, 겹침 자체는 그대로 |
| **DB 자동 번호(시퀀스·AUTO_INCREMENT)** | 겹침이 원천 차단, 대부분 프로젝트의 관례 | 모든 운영 DB에 DDL 반영, 매퍼·저장 순서 변경 |
| 번호 테이블(ID 발급기) | 다중 서버 안전 | 테이블 생성·시작값 맞추기 |
| 테이블 잠금으로 직렬화 | 재시도 없음 | DB마다 문법 다름, 모든 저장이 줄을 섬 |

### 바꿀 때 할 일
- **INSERT에서 키 받기**: Oracle은 INSERT 전에 `NEXTVAL`을 받아 넣고(pre), MySQL은 컬럼을 빼고 INSERT한 뒤 `LAST_INSERT_ID()`로 받는다(post). 같은 커넥션에서 실행돼야 하므로 매퍼의 selectKey나 JDBC `getGeneratedKeys`를 쓴다.
- **저장 순서**: 부모 행과 자식 행을 같은 번호로 미리 만들어 한꺼번에 넣던 코드는, 부모를 먼저 넣고 받은 번호로 자식을 넣게 바꾼다.
- **운영 DB 반영**
  - MySQL: PK 컬럼에 `AUTO_INCREMENT`를 붙이면 시작값은 현재 최대값 다음으로 자동으로 잡힌다. 다른 테이블의 FK가 이 컬럼을 참조하면 `MODIFY`가 ERROR 1833으로 막혀 `FOREIGN_KEY_CHECKS=0`이 필요하다. 테이블을 복사해 다시 만드는 작업이라 행이 많으면 오래 걸리고 그동안 쓰기가 막힌다.
  - Oracle: 시퀀스 시작값을 현재 `MAX+1` 이상으로 만든다. 사이트마다 값이 달라 PL/SQL 블록으로 계산하거나 사람이 넣는다.
- **배포 순서**: 저장 중단 → DB SQL 실행 → 모든 서버에 새 코드 배포 → 재개.
  - 새 코드 + DB 미반영: 저장 실패(Oracle 시퀀스 없음 ORA-02289, MySQL 기본값 없는 PK).
  - 옛 코드 + DB 반영: 옛 코드는 max+1을 직접 넣으므로 정상 동작한다 → 문제가 생기면 코드만 되돌리면 된다.
  - 옛·새 서버가 섞여 도는 롤링 배포: Oracle은 옛 코드가 max+1로 넣은 번호를 시퀀스가 나중에 다시 낼 수 있어 PK 중복 위험. MySQL은 직접 넣은 값만큼 AUTO_INCREMENT가 따라 올라가 덜 위험하다.

## 핵심 코드
### 1) iBATIS/MyBatis 매퍼 — Oracle(pre)
```xml
<insert id="insertResponse" parameterClass="ResponseVO">
    <selectKey keyProperty="responseId" resultClass="long" type="pre">
        SELECT seq_response.NEXTVAL FROM dual
    </selectKey>
    INSERT INTO response (response_id, survey_id, user_id, answer)
    VALUES (#responseId#, #surveyId#, #userId#, #answer#)
</insert>
```

### 2) 매퍼 — MySQL(post)
```xml
<insert id="insertResponse" parameterClass="ResponseVO">
    INSERT INTO response (survey_id, user_id, answer)      <!-- PK 컬럼은 뺀다 -->
    VALUES (#surveyId#, #userId#, #answer#)
    <selectKey keyProperty="responseId" resultClass="long" type="post">
        SELECT LAST_INSERT_ID()
    </selectKey>
</insert>
```

### 3) 저장 순서: 부모를 넣고 받은 번호로 자식
```java
for (ResponseVO r : responses) {
    responseDao.insertResponse(r);              // selectKey가 r.responseId를 채운다
    respondentDao.insert(new RespondentVO(r.getResponseId(), userId));
}
```

### 4) 운영 DB 반영 SQL
```sql
-- MySQL: 참조 FK 때문에 검사를 잠시 끈다(저장을 멈춘 상태에서 실행)
SET FOREIGN_KEY_CHECKS = 0;
ALTER TABLE response MODIFY response_id BIGINT NOT NULL AUTO_INCREMENT;
SET FOREIGN_KEY_CHECKS = 1;
SELECT AUTO_INCREMENT FROM information_schema.tables
 WHERE table_schema = DATABASE() AND table_name = 'response';   -- MAX+1 이상인지 확인
```
```sql
-- Oracle: 시작값을 현재 최대값 다음으로
DECLARE v NUMBER;
BEGIN
  SELECT NVL(MAX(response_id), 0) + 1 INTO v FROM response;
  EXECUTE IMMEDIATE 'CREATE SEQUENCE seq_response START WITH ' || v || ' INCREMENT BY 1 NOCACHE';
END;
/
```

## 주의할 점
- 시퀀스로 바꿔도 "기존 행 삭제 후 재삽입" 구조의 잠금 교착은 별개 문제로 남는다([DELETE 후 INSERT 동시 실행 시 InnoDB 교착(FK 검사 잠금)](DELETE 후 INSERT 동시 실행 시 InnoDB 교착(FK 검사 잠금).md)).
- 여러 쿼리로 저장하는 메서드가 트랜잭션 밖이면, 중간 실패 시 지운 것만 남는다. 번호 문제와 함께 트랜잭션 범위도 확인한다.
- 명령줄 인자로 한글 COMMENT를 넘기면 셸 인코딩 때문에 `????`로 저장될 수 있다. UTF-8 파일로 만들어 `--default-character-set=utf8mb4 < 파일`로 넘긴다.
- 시험: 다른 세션이 다음 번호 행을 커밋하지 않고 잡고 있는 상태에서 저장해 보면, max+1 방식은 잠금을 기다리다 PK 중복으로 실패하고 자동 번호 방식은 바로 성공한다.

## 참고
- MySQL Reference Manual: AUTO_INCREMENT Handling in InnoDB, LAST_INSERT_ID()
- Oracle Database SQL Language Reference: CREATE SEQUENCE
- [AUTO_INCREMENT가 건너뛰며 증가하는 이유](AUTO_INCREMENT가 건너뛰며 증가하는 이유.md)
