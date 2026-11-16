# SQL에 DB 이름(스키마 접두어)을 박아 둘 때 생기는 문제

> 분류: DB·SQL · 최근 갱신: 2026-11-12

## 한 줄 요약
애플리케이션 SQL에 `mydb.users`처럼 DB(스키마) 이름을 직접 쓰면, 같은 코드를 다른 이름의 스키마로 띄웠을 때 그 문장만 엉뚱한 DB를 읽고 쓴다. 접속한 스키마의 테이블을 쓰도록 접두어를 빼고, 남은 곳은 검색으로 찾는다.

## 내용
- **증상**: 시험용으로 스키마를 복제해 `testdb`로 띄웠는데 일부 화면만 원래 DB(`mydb`)의 데이터를 보여 준다. 오류가 없어서 알아채기 어렵다. 원래 DB가 없는 서버에서는 "Unknown database"로 실패한다.
- **DDL 문장도 해당**: `CREATE TABLE ... REFERENCES mydb.parent(id)` 같은 FK는 다른 스키마의 테이블을 가리키게 된다.
- **대소문자**: MySQL은 Linux에서 기본적으로 DB·테이블 이름 대소문자를 구분한다(`lower_case_table_names=0`). `INSERT INTO MYDB.USERS`처럼 대문자로 쓴 문장은 Windows 개발 PC에선 되고 Linux 운영에선 실패할 수 있다.
- **DB 종류별 매퍼를 함께 둔 경우**: 다른 DB용 SQL은 접두어 없이 쓰는데 한 DB용에만 접두어가 있으면 세 DB의 뜻이 달라진다. 같이 대조한다.
- **설정값으로 스키마를 지정하는 곳**(연계 테이블 위치 등)은 일부러 다른 스키마를 가리키는 것일 수 있다. 빈 값이면 접속 스키마를 쓰는지 확인하고, 의도인지 사람에게 묻는다.
- 들어온 시점은 `git log -S'mydb.'`와 `git blame`으로 찾는다. 오래된 공통 코드에서 온 경우가 많아, 파생 저장소만이 아니라 원본 저장소도 같이 고친다.

## 핵심 코드
### 1) 접두어 찾기
```bash
# SQL 파일·매퍼에서 DB 이름 접두어 찾기(대소문자 무시)
grep -rniE '\bmydb\.' --include='*.xml' --include='*.sql' src/
# 처음 들어온 커밋
git log -S'mydb.' --oneline -- src/
```

### 2) 고치기
```sql
-- 전
SELECT COUNT(*) FROM mydb.departments b WHERE b.parent_id = a.id;
INSERT INTO MYDB.PORTLET_SIZE (id, width) VALUES (1, 100);
-- 후: 접속한 스키마의 테이블, 대소문자는 생성문과 같게
SELECT COUNT(*) FROM departments b WHERE b.parent_id = a.id;
INSERT INTO portlet_size (id, width) VALUES (1, 100);
```

### 3) FK가 어느 스키마를 가리키는지 확인 (MySQL)
```sql
SELECT table_schema, table_name, referenced_table_schema, referenced_table_name
  FROM information_schema.key_column_usage
 WHERE referenced_table_name IS NOT NULL
   AND table_schema = DATABASE()
   AND referenced_table_schema <> DATABASE();   -- 다른 스키마를 가리키는 FK
```

## 주의할 점
- 주석·안내 문구·이미 적용된 옛 패치 SQL의 접두어는 실행에 영향이 없으니 그대로 두어도 된다. 실행되는 문장만 고친다.
- 접두어를 빼면 "접속 계정의 기본 스키마"에 의존한다. 커넥션 URL에 스키마가 지정돼 있는지 확인한다.

## 참고
- MySQL Reference Manual — Identifier Case Sensitivity (`lower_case_table_names`)
