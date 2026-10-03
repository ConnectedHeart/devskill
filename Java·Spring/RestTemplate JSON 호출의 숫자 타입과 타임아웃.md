# RestTemplate JSON 호출의 숫자 타입과 타임아웃

> 분류: Java·Spring · 최근 갱신: 2026-10-01

## 한 줄 요약
JSON 라이브러리마다 숫자와 중첩 객체를 다른 자바 타입으로 돌려주기 때문에 캐스팅 코드가 실행 중에 터지고, 호출 하나의 타임아웃만 바꾸려면 전용 `RestTemplate`을 따로 만든다.

## 내용
### 보낼 때: Gson을 한 번 거치면 정수가 실수가 된다
객체를 Gson으로 JSON 문자열로 만들었다가 다시 `Map`/`Object` 계열 타입으로 읽으면 Gson은 숫자를 모두 `Double`로 읽는다. 그 결과를 전송하면 `123`이 `123.0`, 큰 값은 `1.2345678E7`로 나간다. 문자열 `"1"`은 그대로다.
- 받는 쪽이 Jackson으로 `int`/`long` 필드에 받으면 기본 설정에서 대개 통과하지만, `Map`에서 `(Integer)`, `(Long)`으로 캐스팅하면 예외가 난다.
- 본문 객체를 `HttpEntity`에 그대로 넣어 보내면 이런 중간 변환이 없어 정수가 유지된다.

### 받을 때: 파서에 따라 타입이 다르다
| 값 | json-simple (`JSONParser`) | Jackson (`Map`/`Object`로 받을 때) |
|---|---|---|
| 정수 | `Long` | `Integer` (범위를 넘으면 `Long`) |
| 안쪽 객체 | `JSONObject` | `LinkedHashMap` |
| 안쪽 배열 | `JSONArray` | `ArrayList` |

- `(Integer) obj.get("count")`는 컴파일은 되지만 json-simple 결과에서는 `ClassCastException`이다.
- 파서와 무관하게 안전한 형태는 `((Number) obj.get("count")).intValue()`, 문자열로 쓸 값은 `String.valueOf(...)`.
- 응답을 `String.class`로 받아 원하는 파서로 직접 파싱하면 타입이 예측 가능하다.

### 타임아웃
- 연결 대기(connect)와 응답 대기(read)는 별개다. 서버가 살아 있으면 연결은 보통 1초 안에 끝나므로 연결 대기는 짧게(수 초), 응답 대기는 처리 시간에 맞춰 둔다.
- 공용 `RestTemplate` 빈의 타임아웃을 바꾸면 그 빈을 쓰는 모든 호출에 영향이 간다. 특정 호출만 다르게 하려면 전용 인스턴스를 만든다.

## 예시
```java
private final RestTemplate longRestTemplate = createLongRestTemplate();

private static RestTemplate createLongRestTemplate() {
    HttpComponentsClientHttpRequestFactory factory = new HttpComponentsClientHttpRequestFactory();
    factory.setConnectTimeout(5000);
    factory.setReadTimeout(30000);
    return new RestTemplate(factory);
}

// 호출
HttpHeaders headers = new HttpHeaders();
headers.setContentType(MediaType.APPLICATION_JSON);
ResponseEntity<String> res = longRestTemplate.exchange(
        url, HttpMethod.POST, new HttpEntity<>(body, headers), String.class);
JSONObject json = (JSONObject) new JSONParser().parse(res.getBody());
int failCount = ((Number) json.get("failCount")).intValue();
```

## 주의할 점
- 메서드가 호출될 때마다 `HttpComponentsClientHttpRequestFactory`를 새로 만들면 내부 연결 풀이 닫히지 않고 쌓인다. 필드로 한 번만 만들거나 `finally`에서 `factory.destroy()`를 호출한다.
- `RestTemplate` 타입 빈을 하나 더 등록하면 기존의 타입 기반 주입(`@Autowired RestTemplate`)이 모호해져 서버 기동이 실패할 수 있다. 새 빈에 `autowire-candidate="false"`를 주고 이름으로 주입받는다.
- `RestTemplate`은 4xx/5xx 응답에서 예외를 던진다. 호출부에 예외 처리가 필요하다.
- 실패 건이 있을 때만 타는 분기의 캐스팅 오류는 평소에 드러나지 않다가 실패가 생긴 순간에 터진다.
- 오류 응답에는 기대한 필드가 없을 수 있으므로 꺼내기 전에 null을 확인한다.
- 같은 이름의 `JSONArray`/`JSONObject` 클래스가 라이브러리마다 있다(json-simple, org.json, Gson). 직렬화 결과가 다르므로 import를 확인한다.

## 참고
- Spring 문서: RestTemplate, ClientHttpRequestFactory
- Gson 문서: 숫자 역직렬화(ToNumberPolicy)
