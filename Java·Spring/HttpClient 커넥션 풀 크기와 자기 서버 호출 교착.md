# HttpClient 커넥션 풀 크기와 자기 서버 호출 교착

> 분류: Java·Spring · 최근 갱신: 2026-11-04

## 한 줄 요약
Spring RestTemplate을 Apache HttpClient 4.x로 만들 때 풀을 따로 지정하지 않으면 경로(호스트:포트)당 2~5개 연결만 쓰므로, 공유 RestTemplate 하나로 자기 서버나 게이트웨이를 부르면 동시 요청이 몰릴 때 줄을 선다. 풀 크기는 동시 호출 수로 정하고, 자기 서버 호출이면 Tomcat `maxThreads`의 25~40% 이내로 잡아 교착을 막는다.

## 내용
### 기본 풀 크기는 만드는 방법마다 다르다
| 만드는 방법 | 경로당 / 전체 | 비고 |
|---|---|---|
| `new HttpComponentsClientHttpRequestFactory()` (Spring 5.x) | 5 / 10 | 내부에서 `HttpClients.createSystem()` → 시스템 프로퍼티 `http.maxConnections`(기본 5) |
| `HttpClientBuilder.create().build()` | 2 / 20 | `PoolingHttpClientConnectionManager` 기본값 |
| `new RestTemplate()` (JDK `HttpURLConnection`) | 상한 없음 | JDK의 `http.maxConnections`는 유휴 연결 보관 수이지 동시 연결 상한이 아니다. 대신 기본 타임아웃이 없고 PATCH를 지원하지 않는다 |

- 호출마다 팩토리를 새로 만들면 호출마다 자기 풀이 생겨 줄은 서지 않지만, 연결 재사용이 없고 닫지 않은 소켓이 GC 때까지 남는다.
- 그 팩토리를 싱글톤 하나로 바꾸는 순간 WAS 전체가 경로당 5개를 같이 쓰게 되어 병목이 생긴다. "공유로 바꿨더니 몰릴 때 느려졌다"의 흔한 원인이다.
- 풀이 다 찼을 때 기다리는 상한(`connectionRequestTimeout`)도 기본은 무한이다.

### 풀 크기 정하기
- 사용자 수가 아니라 **같은 순간 진행 중인 호출 수**로 정한다: (초당 호출 수) × (호출 1건 소요 시간). 예: 초당 80건 × 0.2초 = 16개.
- 자기 서버 호출(127.0.0.1로 같은 WAS의 API 호출)은 Tomcat 스레드 2개를 쓴다(바깥 요청 + 안쪽 요청). 풀을 `maxThreads`만큼 크게 잡으면 모든 스레드가 바깥 요청으로 묶여 안쪽 요청을 받을 스레드가 없어지고, 읽기 타임아웃까지 서버 전체가 멈출 수 있다. 그래서 경로당 상한은 `maxThreads`의 25~40%(기본 200이면 50).
- `maxThreads`가 아주 크면 기준은 DB 커넥션 풀이 된다. 안쪽 요청도 DB 커넥션을 쓰므로 경로당 상한을 DB 풀보다 충분히 작게 둔다.
- 서버마다 다를 수 있는 값이라 `${키:기본값}`으로 properties에서 읽게 하면 재빌드 없이 서버별로 조정할 수 있다.

### `maxThreads` 확인
- `conf/server.xml`의 HTTP Connector `maxThreads`. `<Executor>`를 쓰면 Executor 값이 쓰이고 Connector 값은 무시된다. 둘 다 없으면 기본 200.
- 실행 중이면 스레드 덤프에서 `http-nio-<포트>-exec-` 스레드 수를 센다. 같은 덤프에서 `leaseConnection`에 묶인 스레드가 있으면 풀 대기 중이다.

### 공유 HttpClient로 바꿀 때 놓치기 쉬운 것
- **쿠키 저장소 공유:** HttpClient는 응답 `Set-Cookie`를 저장했다가 다음 요청에 자동으로 붙인다. 모든 사용자가 클라이언트 하나를 쓰면 A의 호출로 받은 세션 쿠키가 B의 호출에 실린다. `cookieSpec=ignoreCookies`로 끈다(헤더에 직접 넣는 `Cookie`는 그대로 나간다). 단, 상대 서버가 `getSession()`을 부르는 곳이면 호출마다 새 세션이 생긴다.
- **JVM 시스템 설정:** `createSystem()`은 프록시(`http.proxyHost`), `https.protocols`, 암호군, keepAlive 시스템 프로퍼티를 따랐다. 풀을 직접 만들면 이것이 빠지므로 `useSystemProperties()`와 시스템 SSL 소켓 팩토리를 넣는다.
- **우선순위:** `HttpComponentsClientHttpRequestFactory`에 지정한 연결·읽기 타임아웃이 `RequestConfig` 값보다 우선한다. 풀 대기 상한은 `RequestConfig`에서 온다.
- **종료:** `httpClient`에 `destroy-method="close"`를 걸어 재배포 때 소켓을 정리한다.
- 긴 작업(파일 처리 등) 때문에 읽기 타임아웃을 크게 잡았다면, 같은 풀을 쓰되 타임아웃만 다른 RestTemplate을 하나 더 두고 긴 호출만 옮긴다. 느린 호출 하나가 풀 한 자리를 오래 잡지 않게 하기 위해서다.

## 핵심 코드
### 1) XML 빈: 풀 + 풀 대기 상한 + 쿠키 미저장
```xml
<bean id="requestConfigBuilder" class="org.apache.http.client.config.RequestConfig" factory-method="custom">
    <property name="connectTimeout" value="${rest.connectTimeout:10000}" />
    <property name="socketTimeout" value="${rest.socketTimeout:30000}" />
    <property name="connectionRequestTimeout" value="${rest.poolWaitTimeout:30000}" /> <!-- 풀 대기 상한 -->
    <property name="cookieSpec" value="ignoreCookies" />                              <!-- 쿠키 공유 방지 -->
</bean>
<bean id="requestConfig" factory-bean="requestConfigBuilder" factory-method="build" />

<bean id="connectionManager" class="org.apache.http.impl.conn.PoolingHttpClientConnectionManager">
    <property name="maxTotal" value="${rest.maxTotal:200}" />
    <property name="defaultMaxPerRoute" value="${rest.maxPerRoute:50}" />  <!-- maxThreads의 40% 이내 -->
</bean>

<bean id="httpClientBuilder" class="org.apache.http.impl.client.HttpClientBuilder" factory-method="create">
    <property name="defaultRequestConfig" ref="requestConfig" />
    <property name="connectionManager" ref="connectionManager" />
</bean>
<bean id="httpClient" factory-bean="httpClientBuilder" factory-method="build" destroy-method="close" />

<bean id="requestFactory" class="org.springframework.http.client.HttpComponentsClientHttpRequestFactory">
    <constructor-arg ref="httpClient" />
</bean>
<bean id="restTemplate" class="org.springframework.web.client.RestTemplate">
    <constructor-arg ref="requestFactory" />
</bean>
```

### 2) 같은 풀, 다른 타임아웃 (Java)
```java
@Autowired private HttpClient httpClient;   // 위 XML의 공유 클라이언트

public RestTemplate newPooledRestTemplate(int connectMs, int readMs) {
    HttpComponentsClientHttpRequestFactory f = new HttpComponentsClientHttpRequestFactory(httpClient);
    f.setConnectTimeout(connectMs);   // 팩토리 값이 RequestConfig보다 우선
    f.setReadTimeout(readMs);
    return new RestTemplate(f);       // 팩토리는 가볍고 풀은 하나만 쓴다
}
```

### 3) 시스템 프로퍼티(프록시·TLS) 유지가 필요할 때
```java
Registry<ConnectionSocketFactory> reg = RegistryBuilder.<ConnectionSocketFactory>create()
        .register("http", PlainConnectionSocketFactory.getSocketFactory())
        .register("https", SSLConnectionSocketFactory.getSystemSocketFactory())
        .build();
PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager(reg);
cm.setMaxTotal(200);
cm.setDefaultMaxPerRoute(50);
CloseableHttpClient client = HttpClients.custom()
        .useSystemProperties()
        .setConnectionManager(cm)
        .build();
```

### 4) 운영에서 확인
```bash
grep -n -E "<Connector|<Executor|maxThreads|executor=" $CATALINA_BASE/conf/server.xml
jstack <PID> | grep -c '"http-nio-8080-exec-'     # 현재 요청 스레드 수
jstack <PID> | grep -c leaseConnection            # 풀 대기 중인 스레드
```

## 주의할 점
- 다른 코드가 `@PostConstruct` 등에서 XML 빈을 덮어쓰고 있지 않은지 확인한다. 덮어쓰기를 지우기만 하면 기본값(경로당 2)으로 더 나빠질 수 있다.
- 호출부가 아직 `new RestTemplate()`(상한 없음)을 많이 쓰는 상태에서 공유 빈으로만 바꾸면 그 호출이 모두 작은 풀로 몰린다. 풀 설정을 먼저 넣는다.
- 풀 대기 상한을 넣으면 "늦지만 성공"하던 호출이 "오류"로 바뀐다. 모든 호출에 적용되는 동작 변경이다.
- 효과 확인은 같은 부하(동시 N개, 고정 시간)로 처리 건수, p50/p90/p99, 스레드 덤프의 풀 대기 스레드 수를 수정 전후로 비교한다.
- 근본적으로는 같은 WAS 안의 기능을 HTTP로 다시 부르지 말고 서비스 메소드를 직접 호출하는 것이 낫다.

## 참고
- Apache HttpClient 4.5: `PoolingHttpClientConnectionManager`, `HttpClientBuilder.useSystemProperties`, `RequestConfig`
- Spring Framework: `HttpComponentsClientHttpRequestFactory`
- Apache Tomcat Configuration Reference: HTTP Connector, Executor
