# Spring 정적 리소스 캐시 헤더(cache-period)와 버전 쿼리

> 분류: Java·Spring · 최근 갱신: 2026-11-09

## 한 줄 요약
Spring `mvc:resources`에 캐시 기간을 주지 않으면 JS·CSS 응답에 `Cache-Control`이 없어 브라우저가 매번 묻거나 다시 받는다. 파일 주소에 수정 시각 같은 버전 쿼리를 붙이고 있다면 캐시를 길게 잡아도 배포 뒤 옛 파일 문제가 없다.

## 내용
- `<mvc:resources location="/js/" mapping="/js/**"/>`만 있으면 응답에 `Cache-Control: max-age`가 없다. 브라우저는 언제까지 써도 되는지 몰라 조건부 요청(304)을 보내거나 다시 받는다. 요청 하나는 작아도 HTTP/1.1의 서버당 동시 연결 6개 제한과 긴 왕복 시간이 겹치면 연결 차례 대기(Blocked)가 생긴다.
- `cache-period="31536000"`(초)을 주면 `Cache-Control: max-age=31536000`이 붙고, 그 기간 동안 브라우저는 서버에 묻지 않고 캐시에서 바로 쓴다.
- 배포 뒤 새 파일을 받게 하는 방법은 주소를 바꾸는 것이다. 화면 템플릿이 `app.js?v=<수정시각>`처럼 버전을 붙이면 파일이 바뀔 때 주소도 바뀌어 새로 받는다. Spring의 `VersionResourceResolver`(내용 해시를 파일명에 넣음)도 같은 목적이다.
- 버전 없이 직접 참조하는 파일은 캐시 기간 동안 옛 파일이 쓰일 수 있다. 먼저 그런 참조를 찾고, 많으면 `js`·`css`는 길게, 이미지·기타는 짧게(1일) 나눈다.
- 효과를 볼 때는 측정 데이터를 정확히 읽는다. 같은 페이지의 다른 프레임이 이미 받은 파일은 "메모리 캐시"로 0KB가 나온다. 한 번 측정에서 200을 받았다면 "매번 다시 받는다"인지 "처음 받은 것"인지는 가릴 수 없다. 병렬로 받는 작은 스크립트 몇 개는 화면 시간에서 몫이 작을 수 있다(0.1초 안팎 사례).

## 핵심 코드
### 1) XML 설정
```xml
<!-- 버전 쿼리를 붙여 참조하는 정적 파일만 길게 -->
<mvc:resources mapping="/js/**"  location="/js/"  cache-period="31536000"/>
<mvc:resources mapping="/css/**" location="/css/" cache-period="31536000"/>
<mvc:resources mapping="/images/**" location="/images/" cache-period="86400"/>
```

### 2) Java 설정 + 내용 해시 버전
```java
@Override
public void addResourceHandlers(ResourceHandlerRegistry registry) {
    registry.addResourceHandler("/js/**")
            .addResourceLocations("/js/")
            .setCacheControl(CacheControl.maxAge(365, TimeUnit.DAYS))
            .resourceChain(true)
            .addResolver(new VersionResourceResolver().addContentVersionStrategy("/**"));
}
```

### 3) 템플릿에서 버전 붙이기 (예: JSP 함수)
```jsp
<script src="${util:withVersion('/js/list.js')}"></script>
<%-- 결과: /js/list.js?v=20260101120000 (파일 수정 시각) --%>
```

### 4) 버전 없이 참조하는 곳 찾기
```bash
grep -rnE '<script[^>]+src="[^"?]+\.js"' src/main/webapp --include=*.jsp | head
```

## 주의할 점
- 이 설정은 해당 경로의 모든 모듈 정적 파일에 걸린다. 배포 절차에서 버전 값이 실제로 바뀌는지(빌드 산출물의 수정 시각 등) 확인한다.
- 프록시·CDN이 앞에 있으면 그쪽 캐시 규칙도 함께 본다.

## 참고
- Spring Framework 문서: Static Resources, Resource Chain(VersionResourceResolver)
- MDN: Cache-Control, HTTP caching
