# WebSocket 종료 코드로 끊긴 구간 찾기

> 분류: 네트워크 / 웹소켓 · 최근 갱신: 2026-10-27

## 한 줄 요약
브라우저 → 중계 서버 → 백엔드 → 외부 서비스처럼 WebSocket이 여러 단으로 이어질 때, 각 단이 끊는 이유마다 다른 종료 코드와 사유 문자열을 쓰면 브라우저 `onclose`의 `code`/`reason`만 보고 어느 구간에서 끊겼는지 알 수 있다.

## 내용
### 표준 종료 코드 (RFC 6455)
| 코드 | 의미 | 흔한 원인 |
|---|---|---|
| 1000 | 정상 종료 | 클라이언트·서버가 의도적으로 닫음 |
| 1001 | Going Away | 상대가 떠남(서버 종료, 상위 연결 종료를 전달) |
| 1006 | 비정상 종료(프레임 없이 끊김) | 연결 자체 실패, 프록시가 Upgrade 차단, ws/wss 불일치, 네트워크 단절. **직접 보낼 수 없는 코드**로 브라우저가 붙인다 |
| 1008 | Policy Violation | 인증 실패(쿠키·토큰 없음) |
| 1011 | Internal Error | 서버 처리 오류(전송 연속 실패 등) |
| 1013 | Try Again Later | 뒷단 서비스 일시 사용 불가 |

### 다단 중계에서 쓰는 요령
- 각 단에서 실패 원인별로 코드와 짧은 사유(`login required`, `upstream unavailable`)를 정해 닫는다. 브라우저 콘솔에 `closed: <code> <reason>`을 찍으면 1차 구간 판별이 끝난다.
- 뒷단이 끊었을 때 중계 서버가 1001 + 사유로 앞단에 전달하고, 세부 원인은 그 단의 로그 문구로 본다.
- 중계 서버가 뒷단에 붙을 때 짧은 수명의 1회용 티켓(REST로 발급)을 쓰면 쿠키를 넘기지 않아도 된다. 다중 서버라면 티켓 검증이 서버 간 시각·키 차이로 실패할 수 있다.
- 클라이언트가 연결 직후 바로 설정 메시지를 보내는 구조라면, 중계 서버는 `onOpen` 안에서 뒷단 연결까지 끝내 둬야 첫 메시지를 잃지 않는다.
- 화면에서 사용자가 직접 닫은 경우(일시중지·종료)와 서버가 먼저 닫은 경우를 구분해 오류 안내는 후자에만 띄운다.

### 인프라 확인 항목
- 로드밸런서·리버스 프록시가 `Upgrade: websocket`/`Connection: Upgrade`를 통과시키는지
- 유휴 타임아웃이 ping 간격보다 긴지(예: 20초 ping이면 그보다 길게)
- HTTPS 페이지에서는 `wss://`만 허용된다(혼합 콘텐츠)

## 핵심 코드
### 1) 브라우저: 연결 주소와 종료 코드 출력
```javascript
var url = (location.protocol === 'https:' ? 'wss://' : 'ws://') + location.host + '/ws/relay';
var ws = new WebSocket(url);
var closedByUser = false;
function stop() { closedByUser = true; ws.close(1000, 'user stop'); }
setInterval(function () {                       // 유휴 타임아웃보다 짧은 앱 수준 ping
  if (ws.readyState === WebSocket.OPEN) ws.send('{"type":"ping"}');
}, 20000);
```
```javascript
ws.onclose = function (e) {
  console.log('ws closed:', e.code, e.reason);
  if (!closedByUser) showError(); // 사용자가 닫은 게 아닐 때만 안내
};
```

### 2) 서버: 사유와 함께 닫기
```java
// 서버(Jakarta/javax WebSocket)에서 사유와 함께 닫기
session.close(new CloseReason(CloseReason.CloseCodes.VIOLATED_POLICY, "login required"));
```

### 3) 중계 서버: 원인별 코드로 닫고, onOpen 안에서 뒷단까지 연결
```java
@OnOpen
public void onOpen(Session client, EndpointConfig cfg) throws IOException {
    if (cfg.getUserProperties().get("userId") == null) {   // 핸드셰이크에서 넣어 둔 로그인 정보
        client.close(new CloseReason(CloseReason.CloseCodes.VIOLATED_POLICY, "login required"));
        return;
    }
    try {
        connectUpstream(client);                          // 첫 메시지를 잃지 않도록 여기서 끝낸다
    } catch (Exception e) {
        log.warn("upstream connect failed: {}", e.toString());
        client.close(new CloseReason(CloseReason.CloseCodes.getCloseCode(1013), "upstream unavailable"));
    }
}

// 뒷단이 닫히면 앞단에 1001 + 사유로 전달, 세부 원인은 이 로그로 본다
void onUpstreamClose(Session client, CloseReason up) throws IOException {
    log.info("upstream closed code={} reason={}", up.getCloseCode().getCode(), up.getReasonPhrase());
    if (client.isOpen()) {
        client.close(new CloseReason(CloseReason.CloseCodes.GOING_AWAY, "upstream closed"));
    }
}
```

### 4) 리버스 프록시: Upgrade 통과와 유휴 타임아웃 (nginx)
```nginx
location /ws/ {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 120s;      # ping 간격(예: 20초)보다 길게
}
```

### 5) 핸드셰이크만 확인 (101이면 Upgrade 통과)
```bash
curl -i -N --http1.1 \
  -H "Connection: Upgrade" -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Version: 13" -H "Sec-WebSocket-Key: SGVsbG8sIHdvcmxkIQ==" \
  https://example.com/ws/relay
```

## 주의할 점
- 사유 문자열은 123바이트 이하여야 한다. 비밀 값이나 내부 정보를 넣지 않는다.
- 1006은 원인이 가장 넓다. 서버 로그에 연결 기록조차 없으면 프록시·인증서·주소부터 본다.

## 참고
- 기본 용어와 서버·브라우저 문법: [WebSocket 기본 개념과 Java·브라우저 문법](WebSocket 기본 개념과 Java·브라우저 문법.md)
- RFC 6455 7.4 Status Codes
- MDN: CloseEvent.code
