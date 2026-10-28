# L4·L7 로드밸런서와 WebSocket 설정 요청

> 분류: 네트워크 / 웹소켓 · 최근 갱신: 2026-10-27

## 한 줄 요약
WebSocket 기능을 로드밸런서 뒤에 둘 때는 Upgrade 헤더 전달, 유휴 타임아웃(ping 주기보다 넉넉히), 최대 연결 시간 제한 해제, 아웃바운드 허용을 인프라 담당자에게 요청한다. Nginx는 기본이 L7이고 `stream` 모듈로 L4로도 쓸 수 있다.

## 내용
### L4와 L7(Nginx)의 차이
| 구분 | L4 로드밸런서 | Nginx 기본(`http` 블록) |
|---|---|---|
| 보는 정보 | IP, 포트(TCP/UDP) | URL, 헤더, 쿠키, Host |
| 분배 단위 | 연결 | 요청(경로·도메인별 분배 가능) |
| SSL | 대개 통과 | 여기서 종료 가능 |
| 예 | 하드웨어 L4 스위치, AWS NLB, LVS | Nginx, Apache mod_proxy, AWS ALB |
- "ngx"는 Nginx의 줄임말이다(모듈·변수 이름이 `ngx_`로 시작). `stream { upstream ... }`을 쓰면 HTTP를 해석하지 않고 TCP/UDP를 그대로 넘기는 L4로 동작한다.
- L4는 연결 단위 sticky만 가능하고, L7은 쿠키(세션 ID) 기준 sticky나 경로별 분배가 가능하다.

### WebSocket이 끊기는 지점
- **Upgrade 미전달**: 프록시가 `Upgrade: websocket`, `Connection: Upgrade`를 다음 단으로 넘기지 않으면 연결 자체가 안 된다. L4에서 SSL 오프로딩이나 HTTP 단 처리(쿠키·헤더 삽입)를 하면 그 장비도 WebSocket을 지원해야 한다. 사이에 웹서버가 하나 더 있으면 거기도 설정이 필요하다.
- **유휴 타임아웃**: 데이터가 안 흐르면 프록시가 끊는다(Nginx `proxy_read_timeout` 기본 60초). 서버가 주기적으로 ping을 보내고, 장비의 유휴 타임아웃을 ping 주기의 2~3배 이상으로 둔다(예: ping 20초 → 60초 이상).
- **최대 세션 시간**: 회의 녹음처럼 몇 시간 유지되는 연결은 "연결 최대 유지 시간" 설정에 걸려 끊긴다. 해제하거나 업무 최대 시간보다 길게 둔다.
- **sticky**: WebSocket은 한 번 연결되면 그 연결 안에서 끝나므로 필수는 아니다. 다만 연결을 맺는 HTTP 요청에 로그인 쿠키가 전달돼야 한다.
- **서버→다른 서버 구간**: 서버가 외부 서버(예: 음성 인식 서버)로 WebSocket을 다시 여는 구조면 그 아웃바운드도 방화벽·L4에서 허용하고 같은 타임아웃 조건을 맞춘다. 다단 중계면 구간마다 같은 조건이 필요하다.

### 요청문에 넣을 것
1. 연결 구간 표(출발→도착, 주소 경로, 포트)
2. Upgrade 허용, 유휴 타임아웃 하한, 최대 연결 시간, sticky 필요 여부, 아웃바운드 허용
3. 트래픽 추정: 예) 16kHz·16bit·mono PCM 음성 = 16,000×2바이트 = 32KB/s ≈ 256kbps가 연결마다, 동시 사용 수만큼
4. 회신 받을 내용(설정값, 테스트 가능 시점)

### SockJS는 막혀도 대체로 동작
- SockJS(STOMP)는 WebSocket이 막히면 HTTP 스트리밍·폴링으로 자동 전환해 기능이 유지된다. 순수 WebSocket을 쓰는 기능만 막히면 동작하지 않는다. 어떤 기능이 어느 쪽인지 먼저 구분하면 인프라 요청 범위를 줄일 수 있다.
- 단, 클라이언트가 "연결이 끊겨 있으면 주기적으로 페이지를 새로고침"하는 코드를 갖고 있으면 WebSocket이 안 되는 환경에서 입력 중인 내용이 날아갈 수 있다.

## 핵심 코드
### Nginx L7 WebSocket 프록시
```nginx
location /websocket/ {
    proxy_pass http://was_backend;
    proxy_http_version 1.1;                    # Upgrade에는 HTTP/1.1 필요
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_read_timeout 3600s;                  # 유휴 타임아웃: ping 주기보다 충분히 길게
    proxy_send_timeout 3600s;
}
```

### Nginx를 L4로 쓰기
```nginx
stream {
    upstream was_tcp { server 192.0.2.10:8443; server 192.0.2.11:8443; }
    server {
        listen 443;
        proxy_pass was_tcp;
        proxy_timeout 1h;                      # 유휴 타임아웃
    }
}
```

### 서버 쪽 주기 ping (Java, javax.websocket)
```java
ScheduledExecutorService ping = Executors.newSingleThreadScheduledExecutor();
ping.scheduleAtFixedRate(() -> {
    try {
        if (session.isOpen()) session.getBasicRemote().sendPing(ByteBuffer.allocate(0));
    } catch (IOException e) { /* 다음 회차에 다시 시도 */ }
}, 20, 20, TimeUnit.SECONDS);                  // 주기: 장비 유휴 타임아웃의 1/3 이하
```

## 주의할 점
- 장비 앞뒤 구간마다 타임아웃이 따로 있다(L4, 웹서버, WAS 컨테이너). 가장 짧은 값이 실제 한도가 된다.
- ping은 서버→클라이언트 방향만으로도 양방향 유휴 판정을 막는 경우가 많지만, 장비가 방향별로 타이머를 두면 클라이언트 쪽 heartbeat도 필요할 수 있다.
- 업로드 대역폭은 동시 연결 수에 비례한다. 다단 중계면 구간마다 같은 양이 흐른다.

## 참고
- 기본 용어와 서버·브라우저 문법: [WebSocket 기본 개념과 Java·브라우저 문법](WebSocket 기본 개념과 Java·브라우저 문법.md)
- RFC 6455 (The WebSocket Protocol)
- Nginx 문서: WebSocket proxying, ngx_stream_proxy_module
