# LDAP bind로 AD 비밀번호 확인하기

> 분류: 인증·보안 · 최근 갱신: 2026-10-14

## 한 줄 요약
애플리케이션이 Active Directory 비밀번호를 확인할 때는 비밀번호를 비교하지 않고, 입력값으로 AD에 LDAP simple bind를 시도해 성공 여부로 판정한다. 빈 비밀번호·인증서 검증·장애 처리에 주의해야 한다.

## 내용
- **동작**: `ldaps://<AD 서버>:636`에 `<아이디>@<도메인>`(UPN)과 사용자가 입력한 평문 비밀번호로 bind한다. bind가 되면 맞는 비밀번호, 예외가 나면 틀린 것으로 본다. 애플리케이션은 AD의 비밀번호를 알 필요가 없다.
- **평문이 필요함**: bind하려면 원래 비밀번호가 있어야 하므로, 로그인 경로에서 해시로 바꾸기 전의 평문을 AD 확인 로직에 넘기게 된다.
- **비밀번호 동기화 패턴**: bind 성공 시 앱 DB(와 메일 등 연동 시스템)의 비밀번호를 입력값으로 덮어써서, AD에서 바꾼 비밀번호를 다음 로그인 때 따라가게 하는 구현이 흔하다. 이 경우 AD가 기준이고 앱 DB는 사본이다. bind 실패 시에는 DB 비밀번호가 맞아도 거부해야 일관된다.
- **관리용 예외 계정**: AD에 없는 내장 관리자 계정은 bind를 건너뛰고 앱 DB로만 확인하도록 분기하는 경우가 많다.
- **API 로그인 경로**: 화면 로그인 외에 토큰 발급 API를 따로 만들면 같은 AD 확인을 거쳐야 AD 사용자가 로그인할 수 있다. 호출자는 AD 여부와 무관하게 같은 파라미터(아이디·비밀번호)를 보내면 되고, 도메인 접미사는 서버가 붙인다(앱 아이디와 AD 계정명이 같다는 전제).

## 예시
```java
Hashtable<String, String> env = new Hashtable<>();
env.put(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.ldap.LdapCtxFactory");
env.put(Context.PROVIDER_URL, "ldaps://ad.example.com:636");
env.put(Context.SECURITY_AUTHENTICATION, "simple");
env.put(Context.SECURITY_PRINCIPAL, userId + "@example.com");
env.put(Context.SECURITY_CREDENTIALS, password);
try {
    new InitialDirContext(env).close();
    return true;          // 비밀번호 맞음
} catch (AuthenticationException e) {
    return false;         // 비밀번호 틀림
} catch (NamingException e) {
    throw e;              // 서버 장애는 '틀림'과 구분하는 것이 좋다
}
```

## 주의할 점
- **빈 비밀번호**: LDAP simple bind는 비밀번호가 비어 있으면 익명(unauthenticated) bind로 처리되어 "성공"할 수 있다. 빈 값은 bind 전에 반드시 거른다.
- **인증서 검증 생략 금지**: 모든 인증서를 믿는 SSLSocketFactory를 쓰면 통신은 암호화되지만 상대가 진짜 AD인지 확인하지 않는다(중간자 공격 가능). AD의 CA 인증서를 트러스트스토어에 넣는다.
- **장애 = 로그인 실패**: 모든 예외를 "false"로 바꾸면 AD 서버 다운·네트워크 오류에도 비밀번호가 틀린 것으로 처리되어 실패 횟수까지 올라가 계정이 잠길 수 있다. 인증 실패와 연결 실패를 구분한다.
- 동기화 방식이면 토큰 발급 API를 호출할 때도 앱·연동 시스템 비밀번호가 갱신되는 부수 효과가 있다.

## 참고
- Java JNDI LDAP Service Provider 문서
- RFC 4513 (LDAP 인증 방식, unauthenticated bind)
