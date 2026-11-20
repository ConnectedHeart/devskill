# URL·쿼리 문자열 길이 제한(브라우저와 서버별)

> 분류: 네트워크 · 최근 갱신: 2026-11-19

## 한 줄 요약
요즘 브라우저는 URL을 수 MB까지 받으므로, GET 주소가 길어 실패한다면 대개 받는 웹 서버의 요청 줄·헤더 버퍼 제한(약 8KB, IIS 쿼리 2,048자)에 걸린 것이다. 다른 URL을 인코딩해 파라미터로 붙이면 길이가 몇 배로 늘어난다.

## 내용
- 브라우저: Chrome은 약 2MB까지 받는다. 옛 IE의 2,083자 제한은 지금은 거의 의미가 없다.
- 서버 기본값(대략):

| 서버 | 기본 제한 | 넘었을 때 |
|---|---|---|
| Tomcat | 요청 줄 + 헤더 합쳐 8KB(`maxHttpHeaderSize`) | 400 |
| Apache httpd | 요청 줄 8,190바이트(`LimitRequestLine`) | 414 |
| nginx | `large_client_header_buffers` 기본 8KB | 414 |
| IIS | 쿼리 문자열 2,048자(`maxQueryString`), URL 4,096자 | 404.15 |

- 인코딩 증가: `?`·`&`·`/`·`=` 같은 기호는 1자가 `%XX` 3자, 한글은 UTF-8 3바이트라 1자가 9자, 이미 인코딩된 문자열을 다시 인코딩하면 `%`가 `%25`가 되어 한글 1자가 15자가 된다.
- SSO 입구에 `?token=...&returnUrl=<인코딩한 주소>`처럼 붙이는 경우: 쿠키에서 꺼낸 토큰은 쿠키 크기 제한(약 4KB) 안이고, 새로 늘어나는 것은 인코딩된 returnUrl이다. 보통 상세 주소는 수백 자 이내지만 긴 한글 검색어가 붙으면 IIS 기본값을 넘을 수 있다.
- 새 창에서 다른 출처 주소를 열었다가 실패하면 그 창에 상대 서버 오류 화면이 뜰 뿐, 연 쪽 스크립트는 실패를 알 수 없다.

## 핵심 코드
```javascript
// 붙이기 전에 최종 길이를 재고, 상한을 넘으면 대안 경로로 연다. LIMIT은 상대 서버 제한에 맞춘다.
var LIMIT = 2000;
var url = ssoEntry + '?t=' + encodeURIComponent(token) +
          '&returnUrl=' + encodeURIComponent(detailUrl);
if (url.length > LIMIT) {
  url = ssoEntry + '?t=' + encodeURIComponent(token);   // 상세 대신 메인으로
}
```

```xml
<!-- Tomcat server.xml: 헤더·요청 줄 버퍼 늘리기 -->
<Connector port="8080" protocol="HTTP/1.1" maxHttpHeaderSize="16384" />
```

```nginx
# nginx: 요청 줄·헤더 버퍼
large_client_header_buffers 4 16k;
```

## 주의할 점
- 이미 인코딩된 값을 한 번 더 인코딩하지 않는지 먼저 확인한다(길이가 크게 늘고 받는 쪽에서 한 번만 풀면 값도 틀어진다).
- 상대 서버가 무엇인지 모르면 가장 빡빡한 IIS 기준(2,048자)을 상한으로 잡는 것이 안전하다.
- 긴 값은 GET 대신 POST 폼 제출이나 서버에 저장한 짧은 키로 넘기는 방법을 검토한다.

## 참고
- Apache `LimitRequestLine`, nginx `large_client_header_buffers`, Tomcat HTTP Connector `maxHttpHeaderSize`, IIS `requestLimits maxQueryString` 문서
