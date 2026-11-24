# web.xml security-constraint로 http를 https로 리다이렉트하기

> 분류: Java·Spring · 최근 갱신: 2026-11-20

## 한 줄 요약
`web.xml`의 `<transport-guarantee>CONFIDENTIAL</transport-guarantee>`를 건 URL로 http 요청이 오면 서블릿 컨테이너가 https로 리다이렉트하며, 이동할 포트는 소스가 아니라 WAS 설정(Tomcat `redirectPort`)이 정한다.

## 내용
- 서블릿 표준의 `security-constraint`는 인증뿐 아니라 전송 보호 수준도 지정한다. `user-data-constraint/transport-guarantee`가 `CONFIDENTIAL`(또는 `INTEGRAL`)이면 컨테이너는 해당 URL 패턴의 비보안(http) 요청을 보안 연결로 보낸다.
- 리다이렉트 목적지 포트는 애플리케이션이 모른다. Tomcat은 http Connector의 `redirectPort` 값으로 보내고, 다른 WAS는 각자의 설정을 쓴다. https Connector가 없거나 포트가 틀리면 리다이렉트 후 연결이 실패한다.
- 적용 범위는 `url-pattern`에 맞는 주소뿐이다. 로그인 화면 하나만 걸어 두면 비밀번호 전송은 보호되지만 로그인 뒤 다른 화면은 http로 계속 쓸 수 있다. 사이트 전체를 https로 강제하려면 `/*`로 걸거나 앞단(웹 서버·로드밸런서)에서 리다이렉트하고 HSTS를 더한다.
- 코드(Java·필터)에서 https 리다이렉트를 찾을 때 이 설정이 빠지기 쉽다. "https로 넘어가는 곳"을 찾으면 `web.xml`의 `security-constraint`, 웹 서버 설정, 필터를 모두 본다.
- 로컬 개발에서 https Connector가 없으면 로그인 화면이 열리지 않으므로, 이 블록을 주석 처리한 개발용 `web.xml` 사본을 쓰는 경우가 많다.

## 핵심 코드
```xml
<!-- web.xml: 로그인 화면만 https 강제 (사이트 전체면 /* ) -->
<security-constraint>
    <web-resource-collection>
        <web-resource-name>SecureLogin</web-resource-name>
        <url-pattern>/login.do</url-pattern>
    </web-resource-collection>
    <user-data-constraint>
        <transport-guarantee>CONFIDENTIAL</transport-guarantee>
    </user-data-constraint>
</security-constraint>
```

```xml
<!-- Tomcat server.xml: http 요청을 보낼 https 포트 -->
<Connector port="80" protocol="HTTP/1.1" redirectPort="443" />
<Connector port="443" protocol="org.apache.coyote.http11.Http11NioProtocol"
           SSLEnabled="true" scheme="https" secure="true">
    <!-- 인증서 설정 -->
</Connector>
```

```bash
# 확인: Location 헤더가 https://...:<redirectPort> 인지 본다
curl -sI http://localhost/login.do | grep -i -E "^(HTTP|Location)"
```

## 주의할 점
- 값 철자는 `NONE`·`INTEGRAL`·`CONFIDENTIAL` 셋뿐이다. 오타(`CONFIDENTAL` 등)는 WAS마다 무시하거나 다르게 처리할 수 있어, 한 WAS에서 동작했다고 다른 WAS에서도 동작한다고 볼 수 없다. 표준 철자로 쓴다.
- 앞단 프록시가 TLS를 끝내고 WAS에는 http로 넘기는 구조면, WAS는 요청을 계속 비보안으로 보고 리다이렉트를 반복할 수 있다. 이때는 프록시가 보낸 `X-Forwarded-Proto`를 반영하는 설정(Tomcat `RemoteIpValve`)이 필요하다.
- 로컬 실행을 위해 주석 처리한 블록을 그대로 배포하지 않도록, 개발용 사본을 따로 두는 편이 안전하다.

## 참고
- Jakarta Servlet Specification — Security Constraints / transport-guarantee
- Apache Tomcat HTTP Connector 문서 — `redirectPort`, Remote IP Valve
