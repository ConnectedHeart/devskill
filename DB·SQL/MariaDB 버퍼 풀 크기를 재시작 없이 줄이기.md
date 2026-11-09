# MariaDB 버퍼 풀 크기를 재시작 없이 줄이기

> 분류: DB·SQL · 최근 갱신: 2026-11-06

## 한 줄 요약
개발 PC의 MariaDB가 메모리를 많이 잡으면 `innodb_buffer_pool_size`를 `SET GLOBAL`로 바로 줄이고(연결 유지), 같은 값을 설정 파일에 넣어 재시작 뒤에도 유지한다. 크기는 실제로 캐시에 올라온 데이터량을 보고 정한다.

## 내용
- InnoDB 버퍼 풀은 MariaDB 10.2 이상에서 실행 중에 크기를 바꿀 수 있다. 줄일 때도 서비스 재시작 없이 연결이 끊기지 않는다(청크 단위로 조정되므로 값이 조금 맞춰질 수 있다).
- 정하는 기준: InnoDB 전체 데이터 크기와, 버퍼 풀에서 실제로 쓰고 있는 페이지 수(`Innodb_buffer_pool_pages_data` × 16KB). 쓰는 양보다 조금 크게 잡으면 체감 속도는 그대로다.
- MyISAM을 거의 안 쓰면 `key_buffer_size`도 줄일 수 있다(동적). `aria_pagecache_buffer_size`는 동적으로 못 바꾸므로 설정 파일만 고치고 재시작 때 반영된다.
- 줄인 뒤에는 거의 쓰지 않던 테이블의 첫 조회가 디스크를 읽어 느릴 수 있다. 성능 측정 중이라면 예열 회차를 버린다.

## 핵심 코드
### 1) 현재 값과 실제 사용량
```sql
SELECT @@innodb_buffer_pool_size/1024/1024 AS pool_mb;
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_pages_data';     -- × 16KB = 캐시에 올라온 데이터
SELECT ROUND(SUM(data_length+index_length)/1024/1024) AS innodb_mb
  FROM information_schema.tables WHERE engine='InnoDB';
```

### 2) 바로 줄이기
```sql
SET GLOBAL innodb_buffer_pool_size = 768*1024*1024;   -- 실제 사용량에 맞게
SET GLOBAL key_buffer_size = 16*1024*1024;
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_resize_status';   -- 조정 완료 확인
```

### 3) 재시작 뒤에도 유지 (my.ini / my.cnf)
```ini
[mysqld]
innodb_buffer_pool_size=768M
key_buffer_size=16M
aria_pagecache_buffer_size=32M
```

### 4) Windows에서 비밀번호를 출력하지 않고 접속
```powershell
$env:MYSQL_PWD = $env:MY_DB_PW     # 환경 변수로 넘겨 명령줄·로그에 남기지 않는다
& 'C:\Program Files\MariaDB <버전>\bin\mysql.exe' -u $env:MY_DB_USER -e "SELECT @@innodb_buffer_pool_size"
```

## 주의할 점
- 운영 DB에서는 크기 변경이 성능에 바로 영향을 준다. 이 방법은 개발용 로컬 DB를 전제로 한다.
- 설정 파일만 고치고 재시작하지 않았다면 반영 여부는 재시작 후에야 확인된다.

## 참고
- MariaDB 문서: InnoDB Buffer Pool, Setting Innodb Buffer Pool Size Dynamically
