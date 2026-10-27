# LDAP 연결 끊김 원인 구분

> 분류: 네트워크 · 최근 갱신: 2026-10-27

## 한 줄 요약
`connection has been closed`는 방화벽 차단이 아니라 "TCP는 붙었는데 상대가 응답 전에 끊었다"는 뜻이고, 누가 끊었는지는 패킷의 TTL을 비교해서 알 수 있다.

## 내용
### 오류 메시지로 단계 구분
| 메시지 | 의미 |
|---|---|
| `connect timed out` | 패킷이 버려짐. 방화벽 차단이나 경로 문제 |
| `Connection refused` | 도달했지만 포트가 닫혀 있거나 방화벽이 거부 응답 |
| `connection has been closed` | TCP 연결과 요청 전송은 성공, LDAP 응답 대신 연결이 끊김 |

Java(JNDI)에서는 컨텍스트 생성(연결)은 통과하고 `search()`에서 응답을 기다리다 끊기는 형태로 나타난다.

### 응답 전에 끊기는 원인 후보
1. 디렉터리 서버가 등록되지 않은 출발지 IP의 요청을 끊음(접근 제어)
2. 중간 보안장비(IPS 등)가 세션을 리셋
3. 검색 요청 자체의 거부 — 보통은 끊지 않고 LDAP 에러 코드로 응답한다

### 방화벽을 어느 방향으로 열어야 하나
LDAP 조회를 웹 애플리케이션 서버가 수행한다면 출발지는 사용자 PC가 아니라 **애플리케이션 서버**다. "서버 → LDAP 서버"로 열어야 하고, NAT가 있으면 NAT 후 IP가 상대에게 보인다.

### 끊은 주체 찾기: TTL 비교
패킷을 캡처해 연결 수립 때의 SYN/ACK와 끊을 때의 RST/FIN의 TTL을 비교한다.
- 같으면 실제 서버가 끊은 것
- 다르면 중간 장비가 끊은 것

telnet으로 포트가 열려 보여도 중간 장비가 대신 응답한 것일 수 있다. 정상 연동되는 다른 서버에서 같은 명령을 실행해 비교하면 판단이 쉽다.

## 핵심 코드
### 1) Java(JNDI): 단계별 타임아웃과 실패 단계 로그
```java
Hashtable<String, String> env = new Hashtable<>();
env.put(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.ldap.LdapCtxFactory");
env.put(Context.PROVIDER_URL, "ldap://<LDAP 서버>:389");
env.put("com.sun.jndi.ldap.connect.timeout", "5000");   // 연결 단계 → connect timed out
env.put("com.sun.jndi.ldap.read.timeout", "10000");     // 응답 대기 단계

DirContext ctx;
try {
    ctx = new InitialDirContext(env);                    // 실패 = TCP 연결 또는 bind 단계
} catch (NamingException e) {
    log.warn("ldap stage=connect error={} root={}", e.getClass().getSimpleName(), e.getRootCause());
    throw e;
}
try {
    SearchControls sc = new SearchControls();
    sc.setSearchScope(SearchControls.SUBTREE_SCOPE);
    NamingEnumeration<SearchResult> rs = ctx.search("dc=example,dc=com", "(uid=user1)", sc);
} catch (CommunicationException e) {                     // connection has been closed 등
    log.warn("ldap stage=search closed-before-response root={}", e.getRootCause());
    throw e;
} finally {
    ctx.close();
}
```

### 2) 연결 확인 명령
```bash
nslookup <LDAP 서버>
telnet <LDAP 서버> 389
ldapsearch -x -H ldap://<LDAP 서버>:389 -b "" -s base     # -d 1 을 붙이면 단계별 출력

# 리눅스 캡처
tcpdump -i any -nn -v host <LDAP 서버> and port 389 -w ldap.pcap

# 윈도우 캡처
pktmon start --capture
pktmon stop
pktmon etl2pcap PktMon.etl
```

### 3) Windows 포트 확인과 캡처 범위 줄이기
```powershell
Test-NetConnection <LDAP 서버> -Port 389      # TcpTestSucceeded 확인
pktmon filter add -p 389                       # 389 포트만 캡처
pktmon start --capture
# ... 재현 ...
pktmon stop
pktmon etl2pcap PktMon.etl -o ldap.pcapng
pktmon filter remove
```

### 4) 끊은 주체 찾기: SYN/ACK와 RST/FIN의 TTL 비교
```bash
# -v 출력의 ttl 값을 연결 수립 패킷과 끊는 패킷끼리 비교 (같으면 서버, 다르면 중간 장비)
tcpdump -nn -v -r ldap.pcap 'tcp[tcpflags] & (tcp-syn|tcp-fin|tcp-rst) != 0'

# tshark가 있으면 표로
tshark -r ldap.pcap -Y "tcp.flags.syn==1 || tcp.flags.fin==1 || tcp.flags.reset==1" \
  -T fields -e frame.time_relative -e ip.src -e ip.ttl -e tcp.flags
```

## 주의할 점
- 한 애플리케이션이 용도별로 다른 LDAP 연결을 쓰는 경우가 있다(예: 로그인용 LDAPS 636, 조회용 일반 LDAP 389 익명). 어느 연결의 오류인지부터 구분한다.
- 서버 스택 트레이스의 줄 번호가 로컬 소스와 일정하게 어긋나면 버전 차이일 뿐 같은 코드일 가능성이 높다. 차이가 모든 프레임에서 같은지 확인한다.
- 상대 기관의 접근 제어에 내 IP가 등록됐는지는 내 쪽에서 조회할 수 없다. 출발지 IP, 목적지와 포트, 용도를 정리해 상대 운영 주체에 확인한다.

## 참고
- OpenLDAP: ldapsearch
- Microsoft 문서: pktmon
