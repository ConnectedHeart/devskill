# Spring에 선택적 두 번째 데이터소스 추가하기

> 분류: Java·Spring · 최근 갱신: 2026-11-02

## 한 줄 요약
본 DB 말고 다른 DB(외부 제품 DB 등)에 가끔 써야 할 때는, properties 값으로 직접 연결하는 데이터소스를 `lazy-init`·빈 기본값으로 하나 더 두고, URL이 비어 있으면 기능을 끄는 방식으로 만들면 키가 없는 사이트도 그대로 기동된다.

## 내용
### 언제 쓰나
- 다른 서버를 HTTP로 거쳐 그 서버가 DB에 넣던 일을, 중계 없이 직접 넣고 싶을 때. 중계 서버 호출 1번마다 생기던 조회·연결 비용이 사라진다.
- 모든 설치처가 그 DB를 쓰는 것은 아닐 때. 설정이 없는 곳은 기존 경로(중계 서버)를 그대로 타야 한다.

### JNDI와 properties 직접 연결
- **JNDI**: WAS 설정(`context.xml` 등)에 리소스를 등록하고 이름으로 찾는다. 설치처마다 WAS 설정을 고쳐야 한다.
- **properties 직접 연결**: `BasicDataSource`(Commons DBCP) 같은 풀에 URL·계정을 placeholder로 넣는다. 본 DB를 properties로 설정하는 프로젝트라면 같은 방식이 관리하기 쉽다.

### 기동을 깨지 않게 하는 장치
- placeholder에 **빈 기본값**(`${secondary.db.url:}`)을 둔다. 키가 없어도 placeholder 해석이 실패하지 않는다.
- 빈에 **`lazy-init="true"`**를 둔다. 실제로 쓰기 전에는 풀을 만들지 않으므로, URL이 비어 있어도 기동 때 연결 오류가 나지 않는다.
- 쓰는 쪽에서 **URL이 비어 있으면 기능을 끈다**(`isEnabled()`). 끄면 기존 경로로 보낸다.
- 두 번째 `DataSource` 빈이 생기면 타입으로 주입받던 곳(`@Autowired DataSource`)이 모호해질 수 있다. 새 빈에 `autowire-candidate="false"`를 두거나 기존 빈을 `primary`로 하고, 새 빈은 `@Qualifier`로 이름을 지정해 받는다.

### DB 종류가 다를 수 있을 때
- 두 번째 DB 종류(MariaDB, Oracle, Tibero 등)를 별도 키로 받지 않고 연결의 `getMetaData().getDatabaseProductName()`으로 판별할 수 있다.
- 풀 검증 쿼리는 여러 DB에서 도는 `select 1 from dual`처럼 고른다(MariaDB·MySQL도 `dual`을 허용).
- 날짜는 SQL 함수(`now()`, `SYSDATE`) 대신 바인드 값으로 넘기면 DB별 SQL 차이가 줄어든다.

### 트랜잭션
- 본 DB 트랜잭션 매니저는 이 데이터소스를 관리하지 않는다. 직접 JDBC로 쓰는 클래스는 본 트랜잭션 밖에서 돌게 두면, 두 번째 DB가 느려도 본 DB 트랜잭션을 붙잡지 않는다.
- 여러 건을 넣을 때는 연결 하나를 얻어 자동 커밋을 끄고 끝에 한 번 커밋한다([JDBC batch와 다건 INSERT, 일부 실패 처리](../DB·SQL/JDBC batch와 다건 INSERT, 일부 실패 처리.md)).

## 핵심 코드
### 1) 데이터소스 빈 (XML 설정)
```xml
<!-- 키가 없어도 기동되도록 빈 기본값 + lazy-init -->
<bean id="dataSource-secondary" class="org.apache.commons.dbcp2.BasicDataSource"
      destroy-method="close" lazy-init="true" autowire-candidate="false">
    <property name="driverClassName" value="${secondary.db.driver:}"/>
    <property name="url"             value="${secondary.db.url:}"/>
    <property name="username"        value="${secondary.db.username:}"/>
    <property name="password"        value="${secondary.db.password:}"/>  <!-- 암호화 placeholder 지원 시 ENC(...) -->
    <property name="maxTotal"        value="5"/>                           <!-- 가끔 쓰므로 작게 -->
    <property name="validationQuery" value="select 1 from dual"/>
</bean>
```

### 2) 쓰는 쪽
```java
@Component
public class SecondaryWriter {
    @Autowired @Qualifier("dataSource-secondary") @Lazy
    private DataSource dataSource;

    @Value("${secondary.db.url:}")
    private String url;

    public boolean isEnabled() { return url != null && !url.trim().isEmpty(); }

    public int insertAll(List<Item> items) throws SQLException {
        try (Connection conn = dataSource.getConnection()) {
            String product = conn.getMetaData().getDatabaseProductName(); // "MariaDB", "Oracle" ...
            conn.setAutoCommit(false);
            int ok = 0;
            try (PreparedStatement ps = conn.prepareStatement(insertSql(product))) {
                for (Item it : items) {
                    try {
                        ps.setString(1, it.getUserId());
                        ps.setTimestamp(2, new Timestamp(System.currentTimeMillis())); // 날짜는 바인드로
                        ok += ps.executeUpdate();
                    } catch (SQLException e) {
                        // 실패한 건만 건너뜀 (문장 단위 롤백)
                    }
                }
            }
            conn.commit();
            return ok;
        }
    }
}
```

### 3) 호출부에서 경로 고르기
```java
if (secondaryWriter.isEnabled()) {
    secondaryWriter.insertAll(items);      // 직접 넣기
} else {
    relayClient.sendOneByOne(items);       // 설정이 없으면 기존 중계 경로
}
```

## 주의할 점
- 접속 정보는 저장소에 추적되는 설정 파일에 두지 않는다. 본 DB 계정과 같은 보호 수준의 파일·암호화 방식을 쓴다.
- 직접 넣기로 바꾸면 중계 서버가 하던 일(대상 펼치기, 사용자 설정 재확인, 값 변환 등)을 빠뜨리지 않았는지 단계별로 대조한다. 빠뜨리면 알림을 꺼 둔 사용자에게 가는 식의 동작 변화가 생긴다.
- 다른 팀 소유 DB라면 테이블에 직접 INSERT하기보다 그쪽이 제공하는 프로시저를 호출하는 편이 스키마·내부 처리 변경에 덜 묶인다.
- `lazy-init` 빈을 다른 빈이 즉시 주입받으면 그 시점에 생성된다. 주입 지점에도 `@Lazy`를 두거나 쓸 때 꺼낸다.

## 참고
- Spring Framework 문서: Lazy-initialized beans, autowire-candidate, PropertySourcesPlaceholderConfigurer 기본값 문법
- Apache Commons DBCP2 BasicDataSource
