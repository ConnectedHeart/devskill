# DELETE 후 INSERT 동시 실행 시 InnoDB 교착(FK 검사 잠금)

> 분류: DB·SQL · 최근 갱신: 2026-11-17

## 한 줄 요약
"기존 행을 지우고 새로 넣는" 저장을 MySQL InnoDB(REPEATABLE READ)에서 동시에 돌리면 `Deadlock found when trying to get lock`이 날 수 있다. 원인이 인덱스 부족 같아 보여도, 부모 행 삭제 때 FK 검사가 자식 인덱스 끝 구간에 거는 공유 간격 잠금과 항상 끝에 들어가는 새 PK의 INSERT가 엇갈리는 경우가 있어, 인덱스를 추가하면 오히려 늘 수 있다.

## 내용
### 증상
- 같은 사용자가 답을 고쳐 다시 저장하는 기능(기존 응답 DELETE → 새 응답 INSERT)에서, 여러 사람이 거의 동시에 다시 저장하면 한쪽이 교착으로 실패한다. 데이터는 롤백되어 깨지지 않지만 사용자는 다시 눌러야 한다.
- 첫 저장(삭제 없음)은 동시에 해도 실패하지 않는다.

### 처음 생각하기 쉬운 원인과 실제
- **첫 추측**: `DELETE ... WHERE survey_id=? AND user_id=?`에서 `user_id` 인덱스가 없어 `survey_id` 범위 전체와 앞뒤 빈 구간(gap)을 잠그고, 이웃한 설문끼리 gap이 겹쳐 교착.
- **실측**: `(survey_id, user_id)` 인덱스를 넣어도 이웃 설문 교착은 그대로였고, 같은 설문의 다른 사용자끼리는 0건 → 8건(40건 중)으로 늘었다.
- **실제 원인**(교착 정보 `SHOW ENGINE INNODB STATUS`로 확인)
  1. 부모 행을 지우면 InnoDB가 "이 행을 참조하는 자식 행이 있나"를 FK로 확인하며 자식 테이블의 FK 인덱스에 **공유 간격 잠금**을 건다.
  2. 방금 저장한 행을 다시 저장하면 지우는 키가 인덱스 맨 끝에 있어 잠금이 맨 끝 구간에 걸린다.
  3. 새 PK(자동 번호든 max+1이든)는 항상 맨 끝에 들어가므로, 다른 트랜잭션의 INSERT가 그 잠금을 기다리고, 서로 기다리다 교착이 된다.
- **인덱스를 넣었더니 나빠진 이유**: 인덱스가 없을 땐 DELETE가 설문 전체를 잠가 두 번째 저장이 그냥 기다려 직렬화됐다. 인덱스로 잠금이 좁아지자 두 저장이 동시에 진행되며 FK 잠금에서 엇갈렸다.
- 실제 운영에서는 기존 응답 키가 대개 오래된 번호라 끝 구간에 걸리는 일이 드물다. "방금 저장한 것을 바로 다시 저장"을 반복한 시험은 최악의 경우다.

### 해결 방법 비교
| 방법 | 효과 | 크기 |
|---|---|---|
| **교착이 나면 트랜잭션 전체를 다시 하기** | 교착은 나지만 사용자에게는 거의 안 보임. InnoDB 오류 문구도 "try restarting transaction" | 작음 |
| 부모 행(설문) `SELECT ... FOR UPDATE`로 줄 세우기 | 같은 설문끼리는 직렬화, 이웃 설문 교착은 남음 | 중간 |
| 기존 행 PK를 조회한 뒤 PK로 삭제 | 삭제 범위는 줄지만 FK 검사 잠금은 남을 수 있음 | 중간 |
| 삭제 대신 기존 행 갱신 | 근본적, 행 수가 바뀔 때 처리 복잡 | 큼 |
| READ COMMITTED | FK 검사 잠금은 남음, 전체 회귀 위험 | 위험 |
| FK 제거 | 무결성 손실 | 비권장 |

## 핵심 코드
### 1) 교착 원인 보기
```sql
SHOW ENGINE INNODB STATUS;            -- LATEST DETECTED DEADLOCK 절: 어느 인덱스에 어떤 잠금(S/X, gap)인지
SELECT @@transaction_isolation;       -- REPEATABLE-READ이면 gap lock이 생긴다
```

### 2) 재현: 두 세션에서 엇갈리게
```sql
-- 세션 A                                  -- 세션 B
BEGIN;                                     BEGIN;
DELETE FROM response WHERE id = 1001;      DELETE FROM response WHERE id = 1002;   -- 각자 최근 행 삭제(FK 검사 S gap)
INSERT INTO response(survey_id, ...) ...;  -- 대기
                                           INSERT INTO response(survey_id, ...) ...;  -- 교착 → 한쪽 롤백
```

### 3) 트랜잭션 바깥에서 저장 전체를 재시도 (Spring)
```java
// 트랜잭션 프록시 밖(컨트롤러 등)에서 서비스 호출을 감싼다: 시도마다 새 트랜잭션
for (int attempt = 1; ; attempt++) {
    try {
        return responseService.saveAll(form);       // 조회·삭제·INSERT 전부가 한 트랜잭션
    } catch (DeadlockLoserDataAccessException e) {  // 교착만 잡는다
        if (attempt >= 3) throw e;
        Thread.sleep(20 + ThreadLocalRandom.current().nextInt(80));   // 짧은 무작위 대기
    }
}
```

## 주의할 점
- 실패한 INSERT 하나만 다시 하면 안 된다. 교착에 진 트랜잭션은 이미 전체가 롤백됐으므로 조회부터 처음부터 다시 해야 한다. 그래서 재시도는 트랜잭션 경계 바깥에 둔다.
- "인덱스 추가 = 잠금 감소"를 가정하지 말고, 경우(같은 대상·이웃 대상, 같은 사용자·다른 사용자)별로 동시 실행 횟수를 정해 전후 실패 건수를 재서 판단한다.
- 트랜잭션이 없던 저장을 트랜잭션으로 묶으면 정합성은 좋아지지만 잠금이 커밋까지 유지되어 이런 교착이 새로 보일 수 있다.

## 참고
- MySQL Reference Manual: Locks Set by Different SQL Statements in InnoDB, Deadlocks in InnoDB, How to Minimize and Handle Deadlocks
- [max+1 ID를 시퀀스·AUTO_INCREMENT로 바꾸기](max+1 ID를 시퀀스·AUTO_INCREMENT로 바꾸기.md)
- [SELECT FOR UPDATE로 다중 서버 중복 갱신 막기](SELECT FOR UPDATE로 다중 서버 중복 갱신 막기.md)
