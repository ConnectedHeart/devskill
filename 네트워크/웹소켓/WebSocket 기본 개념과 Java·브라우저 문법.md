# WebSocket 기본 개념과 Java·브라우저 문법

> 분류: 네트워크 / 웹소켓 · 최근 갱신: 2026-10-28

## 한 줄 요약
WebSocket은 HTTP 요청으로 시작해(핸드셰이크) 그 연결을 계속 열어 두는 양방향 통로다. Java 서버(JSR-356, `javax.websocket`/`jakarta.websocket`)는 열림·메시지·닫힘·오류를 어노테이션 메소드로 받고, 브라우저는 `WebSocket` 객체의 `onopen`·`onmessage`·`onclose`로 짝을 맞춘다.

## 내용
### HTTP와의 차이
| | HTTP | WebSocket |
|---|---|---|
| 연결 | 요청 1개 → 응답 1개로 끝 | 한 번 연결하면 계속 유지 |
| 보내는 쪽 | 항상 클라이언트가 먼저 | 양쪽 모두 아무 때나 |
| 쓰임 | 페이지, 조회 API | 채팅, 실시간 알림, 음성·영상 스트리밍 |

### 용어
- **핸드셰이크(Handshake) / 업그레이드(Upgrade)**: 처음엔 평범한 HTTP GET에 `Upgrade: websocket`, `Connection: Upgrade` 헤더를 붙여 보내고, 서버가 `101 Switching Protocols`로 답하면 같은 TCP 연결이 WebSocket으로 바뀐다. **쿠키·헤더는 이 첫 요청에만 실려 온다.** 인증 정보는 이 단계에서 꺼내 둬야 한다.
- **엔드포인트(Endpoint)**: 연결을 받는 서버 주소이자 클래스. Spring의 `@Controller` + `@RequestMapping`과 비슷하다. 주소는 `ws://`(평문), `wss://`(TLS). https 페이지에서는 `wss://`만 된다.
- **세션(`javax.websocket.Session`)**: 연결 하나를 나타내는 객체. `HttpSession`과는 다른 것이다. 보내기(`getBasicRemote()`/`getAsyncRemote()`), 닫기(`close()`), 설정(`setMaxIdleTimeout` 등)을 여기서 한다.
- **메시지 종류**: 텍스트(문자열, 보통 JSON)와 바이너리(바이트, 음성·파일 조각) 두 가지뿐이다. 서버는 `@OnMessage` 메소드의 **파라미터 타입**(`String` / `ByteBuffer`·`byte[]`)으로 어느 쪽을 받을지 정한다. 타입별로 하나씩만 둘 수 있다.
- **프레임(Frame)**: 메시지가 실제로 쪼개져 오가는 단위. 데이터 프레임(텍스트·바이너리) 외에 제어 프레임(ping, pong, close)이 있다.
- **ping / pong**: "살아 있니?"/"응" 제어 프레임. 브라우저는 ping을 받으면 자동으로 pong을 보낸다(JS에서 직접 ping을 보낼 API는 없다). 데이터가 안 흐르면 중간 프록시가 연결을 끊는데, ping도 트래픽이라 유휴 타이머를 되감는다.
- **Close와 종료 코드**: 닫을 때 숫자 코드(1000 정상, 1001 떠남, 1008 정책 위반, 1011 서버 오류, 1013 나중에 재시도 등)와 짧은 사유를 보낸다. 브라우저는 `onclose`의 `event.code`/`event.reason`으로 받는다. 코드별 의미와 구간 판별 요령은 [WebSocket 종료 코드로 끊긴 구간 찾기](WebSocket 종료 코드로 끊긴 구간 찾기.md).

### 서버 생명주기 (JSR-356)
`핸드셰이크 → @OnOpen(1번) → @OnMessage(여러 번) → @OnClose(1번)`, 통신 중 예외는 `@OnError`. 상대가 닫든 내가 닫든 `@OnClose`는 불린다.

알아 둘 점:
- **연결마다 엔드포인트 객체가 새로 생긴다**(기본 동작). 인스턴스 필드에 연결별 상태를 둬도 다른 사용자와 섞이지 않는다. 싱글턴인 Spring 컨트롤러와 반대다.
- **컨테이너(Tomcat 등)가 만든 객체라 Spring `@Autowired`가 동작하지 않는다.** 루트 컨텍스트에서 빈을 직접 꺼내거나(`ContextLoader.getCurrentWebApplicationContext()`), `configurator = SpringConfigurator.class`(spring-websocket)를 쓴다.
- **Configurator**: `ServerEndpointConfig.Configurator`를 상속해 핸드셰이크에 끼어드는 훅. `modifyHandshake`에서 쿠키·헤더를 꺼내 `getUserProperties()`에 넣어 두면 `@OnOpen`에서 꺼내 쓸 수 있다. `checkOrigin`으로 Origin 검사도 한다.
- **등록**: 일반 WAR는 컨테이너가 `@ServerEndpoint` 클래스를 스캔해 자동 등록한다(API jar는 `provided`). Spring Boot 내장 톰캣은 `ServerEndpointExporter` 빈이 필요하다.

### 보내기와 설정
- `getBasicRemote().sendText()`: 다 보낼 때까지 기다리는 동기 전송.
- `getAsyncRemote().sendText()`/`sendPing()`: 기다리지 않는 비동기 전송. **이전 전송이 끝나기 전에 또 보내면 `IllegalStateException`**이 난다. 여러 스레드(예: 수신 처리 스레드와 ping 스케줄러)가 같은 세션에 보내면 직렬화하거나 예외를 잡아 건너뛴다.
- `setMaxTextMessageBufferSize`/`setMaxBinaryMessageBufferSize`: 메시지 하나의 최대 크기. 넘으면 1009(Too Big)로 끊긴다. 큰 바이너리를 받으면 올려 둔다.
- `setMaxIdleTimeout`: 이 시간 동안 아무것도 안 오가면 닫는다. 0이나 음수는 무제한. 컨테이너 기본값에 맡기지 말고 정해 두는 편이 안전하다.

### 브라우저 쪽
- `new WebSocket(url)` 생성 즉시 연결을 시작한다. `readyState`: 0 CONNECTING, 1 OPEN, 2 CLOSING, 3 CLOSED. **OPEN 전에 `send()`하면 예외**이므로 첫 메시지는 `onopen` 안에서 보낸다.
- `binaryType = 'arraybuffer'`로 두면 바이너리를 `ArrayBuffer`로 받는다(기본은 `Blob`).
- `send(문자열)` → 텍스트 메시지, `send(ArrayBuffer/TypedArray/Blob)` → 바이너리 메시지.
- `bufferedAmount`: 아직 네트워크로 못 나간 바이트 수. 실시간 스트림에서 계속 커지면 보내는 속도를 줄인다.

### SockJS·STOMP와의 관계
- **순수 WebSocket**: 위에서 설명한 방식. 메시지 형식은 직접 정한다.
- **STOMP**: WebSocket 위에 "구독(subscribe)할 주소"와 "발행(send)" 규칙을 얹은 메시징 프로토콜. Spring에서는 `SimpMessagingTemplate.convertAndSend("/topic/...")`으로 그 주소를 구독한 모두에게 보낸다. 기본 simple broker는 서버 메모리에 있어 여러 WAS 사이에는 전달되지 않는다(외부 브로커 필요).
- **SockJS**: WebSocket이 막힌 환경에서 HTTP 스트리밍·폴링으로 대신 연결해 주는 라이브러리. 인프라 쪽 차이는 [L4·L7 로드밸런서와 WebSocket 설정 요청](L4·L7 로드밸런서와 WebSocket 설정 요청.md).

## 핵심 코드
### 1) 서버 엔드포인트 (javax.websocket, Tomcat 9 이하 / Tomcat 10+는 `jakarta.websocket`)
```java
@ServerEndpoint(value = "/ws/chat", configurator = AuthConfigurator.class)
public class ChatEndpoint {
    private String userId;                       // 연결마다 새 인스턴스 → 연결별 상태

    @OnOpen
    public void onOpen(Session session, EndpointConfig cfg) throws IOException {
        session.setMaxIdleTimeout(30 * 60 * 1000L);
        session.setMaxBinaryMessageBufferSize(4 * 1024 * 1024);
        userId = (String) cfg.getUserProperties().get("userId");
        if (userId == null) {
            session.close(new CloseReason(CloseReason.CloseCodes.VIOLATED_POLICY, "login required"));
        }
    }

    @OnMessage                                    // 텍스트 메시지
    public void onText(String message, Session session) throws IOException {
        session.getBasicRemote().sendText("echo: " + message);
    }

    @OnMessage                                    // 바이너리 메시지
    public void onBinary(ByteBuffer data, Session session) { /* 처리 */ }

    @OnClose
    public void onClose(Session session, CloseReason reason) { /* 자원 정리 */ }

    @OnError
    public void onError(Session session, Throwable t) { /* 로그, 자원 정리 */ }
}
```

### 2) 핸드셰이크에서 쿠키 꺼내기 (Configurator)
```java
public class AuthConfigurator extends ServerEndpointConfig.Configurator {
    @Override
    public void modifyHandshake(ServerEndpointConfig cfg, HandshakeRequest req, HandshakeResponse res) {
        String token = readCookie(req.getHeaders().get("cookie"), "session_token");
        cfg.getUserProperties().put("userId", lookupUserId(token));   // 쿠키 → 사용자 확인
    }

    @Override
    public boolean checkOrigin(String origin) {                         // 다른 사이트에서 오는 연결 차단
        return "https://example.com".equals(origin);
    }
}
```

### 3) 엔드포인트에서 Spring 빈 쓰기
```java
UserService userService = ContextLoader.getCurrentWebApplicationContext()
        .getBean(UserService.class);              // @Autowired 대신 직접 꺼냄
```

### 4) 서버에서 주기 ping (비동기 전송 충돌은 건너뜀)
```java
ScheduledExecutorService ping = Executors.newSingleThreadScheduledExecutor();
ping.scheduleAtFixedRate(() -> {
    try {
        if (session.isOpen()) session.getAsyncRemote().sendPing(ByteBuffer.allocate(0));
    } catch (IllegalStateException | IOException e) { /* 전송 중이면 다음 회차에 */ }
}, 20, 20, TimeUnit.SECONDS);
// @OnClose / @OnError 에서 ping.shutdownNow();
```

### 5) 브라우저
```javascript
var ws = new WebSocket((location.protocol === 'https:' ? 'wss://' : 'ws://') + location.host + '/app/ws/chat');
ws.binaryType = 'arraybuffer';

ws.onopen    = function ()  { ws.send(JSON.stringify({ type: 'hello' })); }; // OPEN 뒤에 첫 메시지
ws.onmessage = function (e) {
  if (typeof e.data === 'string') console.log(JSON.parse(e.data));          // 텍스트
  else console.log('binary', e.data.byteLength);                             // ArrayBuffer
};
ws.onclose   = function (e) { console.log('closed', e.code, e.reason); };
ws.onerror   = function ()  { /* 상세 정보 없음, 이어서 onclose가 온다 */ };

ws.send(new Int16Array(1600).buffer);             // 바이너리 전송
ws.close(1000, 'user stop');                       // 정상 종료
```

## 주의할 점
- 쿠키는 브라우저가 자동으로 붙이므로, 다른 사이트 페이지가 우리 서버로 WebSocket을 열어도 로그인 쿠키가 실린다(Cross-Site WebSocket Hijacking). 쿠키로 인증하면 `checkOrigin`이나 별도 토큰으로 출처를 확인한다.
- `modifyHandshake`에서 `getUserProperties()`에 넣은 값이 연결별로 분리되는지는 컨테이너 구현에 따른다(Tomcat은 연결별 사본). 공유되는 컨테이너면 여러 연결이 동시에 들어올 때 값이 섞일 수 있다.
- 엔드포인트 인스턴스 필드는 연결별이지만 `static` 필드는 서버 전체 공유다. 서버가 여러 대면 다른 서버의 연결은 보이지 않는다.
- `onerror`는 원인을 알려 주지 않는다. 원인은 `onclose`의 코드와 서버 로그로 본다.
- `@OnClose`·`@OnError` 둘 다에서 스케줄러·외부 연결 같은 자원을 정리한다. 두 번 불려도 안전하게 만든다.

## 참고
- RFC 6455 (The WebSocket Protocol)
- JSR 356 / Jakarta WebSocket 명세
- MDN: WebSocket API, CloseEvent
- Spring Framework 문서: WebSockets, STOMP
