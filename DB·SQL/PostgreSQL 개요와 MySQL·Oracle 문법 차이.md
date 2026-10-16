# PostgreSQL 개요와 MySQL·Oracle 문법 차이

> 분류: DB·SQL · 최근 갱신: 2026-10-14

## 한 줄 요약
PostgreSQL(줄여서 Postgres, PG)은 표준 SQL을 충실히 지원하는 오픈소스 관계형 데이터베이스로, Oracle 대체로 많이 검토된다. "postgre", "postgral"은 흔한 오기다.

## 내용
- 라이선스 비용이 없는 오픈소스 RDBMS.
- 트랜잭션, 외래키, 복잡한 조인·서브쿼리, 윈도우 함수, CTE(`WITH`)를 잘 지원해 Oracle과 쓰임새가 비슷하다.
- `jsonb`, 배열 타입, 전문 검색, 사용자 정의 함수, 확장(PostGIS 등)이 강점이다.
- 주요 클라우드(AWS, Azure, GCP)에서 관리형으로 제공된다.
- 여러 DB를 지원하는 애플리케이션이 DB별 SQL 파일(MyBatis/iBATIS sqlmap 등)을 두는 구조라면, PostgreSQL 지원은 모든 SQL 파일에 PG 버전을 추가하는 큰 작업이 된다.

## 예시
| 항목 | MySQL | Oracle | PostgreSQL |
|---|---|---|---|
| 문자열 합치기 | `CONCAT(a, b)` | `a \|\| b` | `a \|\| b` (`CONCAT`도 가능) |
| 행 수 제한 | `LIMIT 10` | `ROWNUM` / `FETCH FIRST` | `LIMIT 10` / `FETCH FIRST` |
| 자동 증가 | `AUTO_INCREMENT` | 시퀀스 / IDENTITY | `SERIAL` / IDENTITY / 시퀀스 |
| NULL 대체 | `IFNULL` | `NVL` | `COALESCE` |

## 주의할 점
- PostgreSQL은 따옴표 없는 식별자를 소문자로 바꾼다. 대문자 테이블명을 쓰던 스키마를 옮길 때 주의한다.
- 문자열과 숫자 비교 등 암묵적 형변환이 MySQL보다 엄격하다.

## 참고
- PostgreSQL 공식 문서 (postgresql.org/docs)
