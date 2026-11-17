# DB 작업 큐를 스케줄러로 처리할 때 막히는 지점

> 분류: Java·Spring · 최근 갱신: 2026-11-16

## 한 줄 요약
작업 행을 READY → ING → END로 바꾸며 `@Scheduled`로 처리하는 알림 큐는, 한 건 예외로 전체 실패, 재기동 시 ING 고착, 긴 작업이 다음 실행을 미룸, 트랜잭션 하나로 묶여 앞부분이 늦게 보임 같은 문제가 생기기 쉽다. 수신자·채널별 예외 격리, 조건부 선점, 실제 완료 후 END, 트랜잭션 분리로 막는다.

## 내용
### 자주 생기는 문제
- **한 명·한 채널 오류로 작업 전체 실패**: 수신자 반복문 안에서 사용자 조회 null, 날짜 형식 파싱 실패 같은 예외가 나면 재시도 한도 후 FAIL로 끝나고 아무에게도 안 간다. 예: 종료일을 `yyyy-MM-dd HH:mm`으로만 파싱하는데 날짜만 오는 정상 입력에서 ParseException → 메일을 끈 경우에도 메일 본문을 먼저 만들다 실패.
- **ING 고착**: 고른 작업을 한꺼번에 ING로 바꾼 뒤 하나씩 처리하면, 도중 재기동 때 손대지 않은 작업까지 ING로 남는다. 조회 조건이 `READY`·`FAIL`뿐이면 다시는 처리되지 않는다. ING가 있다고 다른 작업이 막히지는 않는다.
- **긴 작업이 다음 실행을 미룸**: Spring은 같은 `@Scheduled` 메소드를 겹쳐 돌리지 않는다. 수신자 판정(전 직원 × 권한 확인)이 스케줄러 스레드에서 오래 걸리면 그동안 새 작업이 READY로 쌓인다. READY의 생성 시각과 ING의 처리 시각을 같이 보면 고착과 지연을 구분할 수 있다.
- **END가 "완료"가 아님**: 비동기 발송에 넘기자마자 END로 바꾸면 END인데 알림이 없을 수 있다.
- **느린 채널이 빠른 채널을 붙잡음**: 수신자마다 메일(SMTP)을 먼저 동기로 보내면 메일 서버 시간 초과(20초)만큼 인앱 알림이 밀린다. 순서를 바꿔도 **발송 메소드 전체가 트랜잭션 하나**면 메일까지 끝나야 커밋되어 여전히 늦게 보인다. 그 메소드를 트랜잭션 밖으로 빼 수신자별 INSERT가 바로 커밋되게 해야 효과가 난다.
- 이중화 서버에서 선출이 겹치면 두 서버가 같은 작업을 보낼 수 있다.

### 고치는 방향
1. 수신자 반복문은 사람마다 try/catch, 실패한 사람만 ID와 함께 로그 후 건너뛴다. 내용 만들기는 채널별로 감싸 메일 본문 실패 시 다른 채널은 보낸다. 모든 채널이 실패할 때만 작업 FAIL.
2. 빠른 채널(인앱·푸시)을 모든 수신자에게 먼저 보내고, 메일은 뒤에 묶어서(같은 본문이면 수십 명씩, 받는 사람끼리 주소가 안 보이게 개별발신·BCC) 보낸다.
3. 선점은 조건부 UPDATE로 한 건씩: 바뀐 행이 1건일 때만 처리 → 서버 간 중복도 막힌다.
4. END는 실제 발송이 끝난 뒤. 처리 중에는 몇 명마다 `processed_at`을 갱신(하트비트)하고, N분 넘게 갱신 없는 ING는 다시 가져간다. 재처리 중복을 막으려면 진행 위치 컬럼과 수신자 정렬이 필요하다.
5. 공용 메일 유틸이 연결 실패를 안에서 삼키면 호출부가 "메일 서버 죽음 → 남은 메일 건너뛰기"를 못 한다. 유틸이 결과를 돌려주게 바꿔야 한다.

## 핵심 코드
```sql
-- 한 건씩 선점: 영향 행 1일 때만 처리
UPDATE noti_job
   SET status = 'ING', processed_at = NOW(), process_cnt = process_cnt + 1
 WHERE job_id = ? AND status IN ('READY', 'FAIL') AND process_cnt < 3;

-- 멈춘 ING 다시 가져가기 대상
SELECT job_id FROM noti_job
 WHERE (status IN ('READY','FAIL') AND process_cnt < 3)
    OR (status = 'ING' AND processed_at < NOW() - INTERVAL 10 MINUTE AND process_cnt < 3);
```
```java
// 수신자·채널별 예외 격리 + 빠른 채널 먼저
public void send(Job job, List<String> userIds) {           // 트랜잭션 밖(설정으로 제외)
    List<User> mailTargets = new ArrayList<>();
    for (String id : userIds) {
        try {
            User u = userService.find(id);
            if (u == null) { log.warn("skip missing user {}", id); continue; }
            inAppSender.send(u, job);                         // 각자 바로 커밋
            pushSender.send(u, job);
            mailTargets.add(u);
        } catch (RuntimeException e) {
            log.error("noti failed user={} job={}", id, job.getId(), e);   // 그 사람만 건너뜀
        }
    }
    for (List<User> chunk : partition(mailTargets, 50)) {    // 메일은 마지막에 묶어서
        try { mailSender.sendEach(chunk, job.mailBody()); }
        catch (RuntimeException e) { log.error("mail chunk failed job={}", job.getId(), e); }
    }
}
```
```java
// 날짜만 오는 입력도 받기
String end = raw.length() > 10 ? raw : raw + " 23:59";
Date endDate = new SimpleDateFormat("yyyy-MM-dd HH:mm").parse(end);
```

## 주의할 점
- 일부 수신자를 건너뛰어도 작업 상태는 END로 두고 로그로 추적한다. 상태 규칙까지 바꾸면 기존 화면·통계가 달라진다.
- 트랜잭션에서 빼면 중간 실패 시 일부만 저장된 상태가 남는다. 수신자 단위로 독립적인 작업에만 쓴다.
- 개별발신을 메일 서버가 지원하지 않으면 받는 사람끼리 주소가 보인다. 묶어 보내기 전에 서버 지원을 확인한다.

## 참고
- Spring Framework Reference — Task Execution and Scheduling (@Scheduled)
- [SELECT FOR UPDATE로 다중 서버 중복 갱신 막기](../DB·SQL/SELECT FOR UPDATE로 다중 서버 중복 갱신 막기.md)
- [Spring 트랜잭션 전파와 자동 커밋](Spring 트랜잭션 전파와 자동 커밋.md)
