# RequestBody와 요청 파라미터(getParameterNames)의 차이

> 분류: Java·Spring · 최근 갱신: 2026-10-27

## 한 줄 요약
`@RequestBody`는 JSON 본문만 읽고, `request.getParameterNames()`에는 쿼리스트링과 `x-www-form-urlencoded` 폼 본문이 들어간다. JSON 본문 API에 Params/form으로 값을 보내면 읽히지 않거나 필터에서 거부된다.

## 내용
- `@RequestBody` 객체는 `Content-Type: application/json` 본문을 메시지 컨버터로 역직렬화한 것이다. 같은 이름을 쿼리스트링이나 폼으로 보내도 채워지지 않는다.
- 서블릿의 `getParameter*`는 URL 쿼리스트링과 폼 인코딩 본문(`application/x-www-form-urlencoded`)을 합친다. `multipart/form-data`는 컨테이너·설정에 따라 다르다.
- "핸들러에 `@RequestParam`으로 선언되지 않은 파라미터는 400" 같은 엄격한 인터셉터를 두면, JSON 본문 API에 Postman Params 탭이나 form 탭으로 보낸 요청이 모두 걸린다. 오류가 아니라 정상 동작이며, 보내는 방식을 Body → raw → JSON으로 바꾸면 된다.

## 핵심 코드
### 1) JSON 본문을 받는 핸들러와 호출 방법
```java
@PostMapping("/api/token")
public TokenResponse issue(@RequestBody TokenRequest req) { ... }  // JSON만
```
```bash
# 올바름
curl -X POST http://localhost:8080/api/token -H "Content-Type: application/json" -d '{"username":"u","password":"p"}'
# 읽히지 않음(쿼리스트링)
curl -X POST "http://localhost:8080/api/token?username=u&password=p"
```

### 2) 폼·쿼리스트링으로 받는 핸들러 (대조용)
```java
@PostMapping(value = "/v1/items", consumes = MediaType.APPLICATION_FORM_URLENCODED_VALUE)
public String save(@RequestParam("name") String name, HttpServletRequest request) {
    // 쿼리스트링 + x-www-form-urlencoded 본문이 함께 들어 있다. JSON 본문은 여기 없다.
    for (String p : Collections.list(request.getParameterNames())) {
        log.debug("param: {}", p);
    }
    return "ok";
}
```
```bash
curl -X POST http://localhost:8080/v1/items -d "name=abc"   # -d는 기본이 form-urlencoded
```

### 3) 선언되지 않은 파라미터를 거부하는 인터셉터
```java
public class StrictParamInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler)
            throws Exception {
        if (!(handler instanceof HandlerMethod)) {
            return true;
        }
        Set<String> allowed = new HashSet<>();
        for (MethodParameter mp : ((HandlerMethod) handler).getMethodParameters()) {
            RequestParam rp = mp.getParameterAnnotation(RequestParam.class);
            if (rp != null) {
                String n = !rp.value().isEmpty() ? rp.value() : rp.name();
                if (!n.isEmpty()) {
                    allowed.add(n);           // 이름을 명시한 @RequestParam만 허용 목록에
                }
            }
        }
        for (String name : Collections.list(req.getParameterNames())) {
            if (!allowed.contains(name)) {
                res.sendError(HttpServletResponse.SC_BAD_REQUEST);   // JSON API에 Params로 보내면 여기 걸림
                return false;
            }
        }
        return true;
    }
}
```

## 주의할 점
- 접근 로그 인터셉터가 경로와 쿼리스트링을 남기는 경우가 많다. 비밀번호를 쿼리스트링으로 보내 테스트했다면 서버 로그에 평문으로 남았을 수 있으니 로그 정리나 비밀번호 변경을 고려한다. 본문은 보통 로그에 남지 않는다.

## 참고
- Spring Framework 문서 — @RequestBody, @RequestParam
- Jakarta Servlet 명세 — Request Parameters
