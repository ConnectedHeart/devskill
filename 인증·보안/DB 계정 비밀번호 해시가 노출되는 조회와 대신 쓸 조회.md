# DB 계정 비밀번호 해시가 노출되는 조회와 대신 쓸 조회

> 분류: 인증·보안 · 최근 갱신: 2026-11-11

## 한 줄 요약
MariaDB/MySQL의 `SHOW GRANTS`·`SHOW CREATE USER`·`mysql.user` 조회 결과에는 계정 비밀번호 해시가 들어 있어 로그·대화 기록·화면 공유로 새기 쉽다. 권한 확인은 해시가 없는 `information_schema.*_PRIVILEGES`로 하고, AI 에이전트나 자동화 스크립트에는 이 조회를 하지 않도록 규칙과 차단을 함께 둔다.

## 내용
- 해시가 나오는 조회
  - `SHOW GRANTS FOR ...` (구버전·MariaDB에서 `IDENTIFIED BY PASSWORD '*<40자 16진수>'` 꼴로 붙는다)
  - `SHOW CREATE USER ...`
  - `SELECT * FROM mysql.user` / `mysql.global_priv`
- 해시가 위험한 이유: `mysql_native_password` 해시는 솔트 없는 SHA1 두 번이라 오프라인 대입·사전 공격이 쉽다. 비밀번호가 약하면 평문이 새는 것과 비슷하다. 또 이 방식은 해시만 있어도 인증 과정 일부를 흉내 낼 수 있는 구조라 더 민감하다.
- 피해 범위를 키우는 설정: DB가 `0.0.0.0`(모든 인터페이스)에서 열려 있고, 계정이 `'user'@'%'`(모든 호스트)에 SUPER·GRANT OPTION까지 가진 경우. 같은 망의 다른 PC에서 바로 접속을 시도할 수 있다.
- 새었을 때 할 일
  1. 어디에 남았는지 찾는다: 셸 기록, 앱·에이전트 대화 기록 파일, 로그, 공유 폴더. 해시 형식(`*` + 16진수 40자)으로 검색한다.
  2. 값만 가린다(`****`). 원본이 남는 백업은 만들지 않는다. 이미 외부 서비스로 전송된 것은 되돌릴 수 없다.
  3. 비밀번호를 바꾸고, 그 비밀번호를 쓰는 곳(환경 변수, 앱 설정, 복사본 설정)을 함께 바꾼다.
  4. 노출 범위를 줄인다: 로컬 전용이면 `bind-address=127.0.0.1`, 계정 호스트를 `localhost`로 좁힘, 앱 계정에서 관리 권한 제거.
- 재발 방지: 지침(메모리·규칙 문서)은 "하지 말라"는 안내일 뿐 실행을 막지 못한다. 확실히 막으려면 명령 실행 전 훅이나 권한 규칙으로 해당 문자열이 든 명령을 거부한다. 또 관리자 계정으로는 계정·권한·접속 설정 같은 중요 정보를 조회하지 않고, 검증용 조회는 애플리케이션 계정으로 업무 테이블만 한다.

## 핵심 코드
### 1) 해시 없이 권한 확인
```sql
SELECT GRANTEE, PRIVILEGE_TYPE, IS_GRANTABLE
  FROM information_schema.USER_PRIVILEGES   WHERE GRANTEE LIKE '''appuser''@%';
SELECT GRANTEE, TABLE_SCHEMA, PRIVILEGE_TYPE
  FROM information_schema.SCHEMA_PRIVILEGES WHERE GRANTEE LIKE '''appuser''@%';
SELECT GRANTEE, TABLE_SCHEMA, TABLE_NAME, PRIVILEGE_TYPE
  FROM information_schema.TABLE_PRIVILEGES  WHERE GRANTEE LIKE '''appuser''@%';
```

### 2) SHOW GRANTS가 꼭 필요할 때 가려서 출력
```bash
mysql -N -e "SHOW GRANTS FOR 'appuser'@'%'" \
  | sed -E "s/(IDENTIFIED BY PASSWORD )'[^']*'/\1'****'/g"
```

### 3) 기록 파일에서 해시 찾기·가리기
```bash
grep -rlE "\*[0-9A-F]{40}" ~/logs ~/.app-history          # 남은 곳 찾기
sed -i -E "s/\*[0-9A-F]{40}/****/g" <찾은 파일>            # 값만 가림, 형식은 유지
```

### 4) 노출 범위 줄이기
```ini
# my.ini / my.cnf  [mysqld]
bind-address = 127.0.0.1
```
```sql
RENAME USER 'appuser'@'%' TO 'appuser'@'localhost';   -- 다른 PC 접속이 필요 없을 때만
ALTER USER 'appuser'@'localhost' IDENTIFIED BY '<새 비밀번호>';
```

## 주의할 점
- JSON 같은 구조화된 기록을 가릴 때는 값만 바꿔 형식이 깨지지 않게 한다.
- 다른 PC가 이 DB에 붙는 구성(공유 개발 DB 등)이 있으면 bind-address·호스트 축소 전에 확인한다.
- 비밀번호를 바꾸면 그 값을 쓰는 모든 설정 사본을 함께 바꿔야 서버가 기동한다.

## 참고
- MariaDB Knowledge Base: SHOW GRANTS, information_schema USER_PRIVILEGES
- MySQL Reference Manual: Native Pluggable Authentication
