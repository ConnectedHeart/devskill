# PL-SQL 패키지 명세·본문과 프로시저 시그니처

> 분류: DB·SQL · 최근 갱신: 2026-11-03

## 한 줄 요약
Oracle(Tibero 포함) 패키지는 명세(스펙, 선언부)와 본문(바디, 구현부)으로 나뉘고, PL/SQL 블록 안은 선언부·실행부·예외처리부로 나뉜다. 프로시저를 부르는 파라미터 규격은 시그니처라고 부른다.

## 내용
### 패키지: 명세와 본문
| 흔히 부르는 말 | 공식 명칭 | 내용 |
|---|---|---|
| 선언부 | 패키지 명세(Package Specification, "스펙") | 외부에 공개하는 프로시저·함수 이름과 파라미터 |
| 구현부 | 패키지 본문(Package Body, "바디") | 실제 로직 |

- Java로 치면 명세가 interface, 본문이 구현 클래스다. 문서에서는 "패키지 스펙 / 패키지 바디"라는 말을 더 많이 쓴다.
- `CREATE PROCEDURE`로 만든 단독 프로시저는 선언과 구현이 나뉘지 않고 한 덩어리다.
- MySQL에는 패키지가 없어서 이렇게 나누지 않는다.

### PL/SQL 블록 구조: 선언부·실행부·예외처리부
- `IS`(또는 `AS`)와 `BEGIN` 사이가 선언부(변수·커서), `BEGIN` 이후가 실행부, `EXCEPTION` 이후가 예외처리부다.
- 이 구조를 말할 때는 "구현부"가 아니라 "실행부"라고 한다.

### 시그니처와 관련 용어
- 시그니처(signature): 프로시저 이름 + 파라미터의 순서·이름·타입·모드(IN / OUT / IN OUT). 개발자끼리 가장 흔히 쓰는 말이다("시그니처가 바뀌었다").
- 파라미터 명세 / 인터페이스 명세: 문서나 시스템 간 연계 정의서에서 쓰는 말.
- 형식 파라미터(formal parameter): 정의에 선언된 이름(`p_doc_id`).
- 실제 파라미터(actual parameter, 인자): 호출할 때 넘기는 값(`'DOC001'`).
- 패키지 명세는 시그니처를 모아 둔 곳이다. "스펙 보고 호출하라"는 시그니처대로 호출하라는 뜻이다.

## 핵심 코드
### 1) 패키지 명세와 본문
```sql
-- 명세(스펙): 시그니처만
CREATE OR REPLACE PACKAGE order_pkg AS
    PROCEDURE get_order (p_order_id IN VARCHAR2, p_result OUT SYS_REFCURSOR);
END order_pkg;
/

-- 본문(바디): 구현
CREATE OR REPLACE PACKAGE BODY order_pkg AS
    PROCEDURE get_order (p_order_id IN VARCHAR2, p_result OUT SYS_REFCURSOR) IS
    BEGIN
        OPEN p_result FOR SELECT * FROM orders WHERE order_id = p_order_id;
    END get_order;
END order_pkg;
/
```

### 2) 단독 프로시저의 블록 구조
```sql
CREATE OR REPLACE PROCEDURE count_orders (p_user_id IN VARCHAR2) IS
    v_cnt NUMBER;                                            -- 선언부
BEGIN
    SELECT COUNT(*) INTO v_cnt FROM orders WHERE user_id = p_user_id;   -- 실행부
EXCEPTION
    WHEN OTHERS THEN RAISE;                                  -- 예외처리부
END;
/
```

### 3) 시그니처 조회
```sql
-- 프로시저 파라미터 순서·타입·모드 확인
SELECT object_name, argument_name, position, data_type, in_out
  FROM user_arguments
 WHERE package_name = 'ORDER_PKG'
 ORDER BY object_name, position;
```

## 주의할 점
- 명세의 시그니처를 바꾸면 그 패키지를 쓰는 다른 객체가 INVALID가 되어 재컴파일이 필요하다. 본문만 바꾸면 호출하는 쪽은 영향을 받지 않는다.

## 참고
- Oracle Database PL/SQL Language Reference: Packages, PL/SQL Block Structure
