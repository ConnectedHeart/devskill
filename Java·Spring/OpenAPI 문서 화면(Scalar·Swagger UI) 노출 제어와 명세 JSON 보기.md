# OpenAPI 문서 화면(Scalar·Swagger UI) 노출 제어와 명세 JSON 보기

> 분류: Java·Spring · 최근 갱신: 2026-11-23

## 한 줄 요약
Scalar·Swagger UI 같은 API 문서 화면은 서버가 내주는 OpenAPI 명세 JSON(`/openapi/v1.json` 등)을 받아 그리는 껍데기라서, 화면 경로와 명세 경로를 따로 막고 열 수 있고, 명세 JSON 하나만 받으면 화면의 모든 내용을 가진 것이 된다.

## 내용
- **두 경로의 역할**: 문서 화면(`/scalar`, `/swagger/**`)은 HTML·JS이고, 내용은 명세 경로(`/openapi/**`, `/v3/api-docs`)에서 읽는다. 화면만 열고 명세를 막으면 화면은 뜨지만 비어 보인다.
- **막는 방법**: 인터셉터나 필터에서 설정값으로 두 경로를 각각 404로 막는다. 화면은 운영에서 기본 차단(설정이 없거나 오타여도 막힘), 명세는 다른 시스템 연동이 읽어 갈 수 있으므로 기본 열림처럼 기본값을 다르게 둘 수 있다. 이때 실제 API(`/v1/**`) 호출은 영향이 없어야 한다.
- **설정 반영 시점**: properties를 기동 때 읽는 구조면 값을 바꾼 뒤 재기동해야 한다. 빌드 산출물 쪽 설정 파일에도 바뀐 값이 들어갔는지 본다.
- **외부 CDN 의존**: Scalar는 보통 CDN(jsdelivr 등)에서 스크립트를 받아 그린다. 외부망이 막힌 환경에서는 화면이 비고, 서버 안 정적 파일만 쓰는 Swagger UI 묶음은 그대로 동작한다.
- **API 시험**: 토큰 발급 API를 화면에서 먼저 불러 받은 값을 Authorize 칸에 넣는다(Bearer 접두어는 화면이 붙이는 경우가 많음).
- **명세 받기**: 화면의 "Download OpenAPI Document" 링크나 명세 URL을 직접 받는다. 경로·파라미터·요청·응답 스키마·예시가 다 들어 있고 Postman Import·Swagger Editor에 그대로 넣을 수 있다. 업무 데이터는 들어 있지 않다.
- **JSON 보기**: 사내 명세는 온라인 JSON 뷰어에 올리지 않는다. Firefox 내장 JSON 뷰어(트리·검색·필터)가 가장 편하고, Edge는 자동 정리·접기, Chrome은 "Pretty-print" 체크박스를 쓴다. Notepad++는 플러그인 관리자에서 JsonTools(큰 파일·검색), JSON Viewer, JSTool 중 하나를 설치한다(트리 보기 단축키가 겹치므로 하나만).

## 핵심 코드
```java
// 문서 화면과 명세 경로를 설정값으로 따로 막는 인터셉터 (Spring MVC)
public class ApiDocsInterceptor implements HandlerInterceptor {
    @Value("${app.apiDocs.ui:NO}")   private String uiOpen;    // 없으면 차단
    @Value("${app.apiDocs.spec:YES}") private String specOpen; // 없으면 열림

    @Override
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object h) throws IOException {
        String path = req.getRequestURI().substring(req.getContextPath().length());
        boolean ui   = path.startsWith("/scalar") || path.startsWith("/swagger");
        boolean spec = path.startsWith("/openapi") || path.startsWith("/v3/api-docs");
        if ((ui && !"YES".equalsIgnoreCase(uiOpen)) || (spec && "NO".equalsIgnoreCase(specOpen))) {
            res.sendError(HttpServletResponse.SC_NOT_FOUND);
            return false;
        }
        return true;
    }
}
```

```properties
# 로컬 확인용: 둘 다 열어야 화면에 내용이 보인다
app.apiDocs.ui = YES
app.apiDocs.spec = YES
```

```powershell
# 명세 JSON 받기
Invoke-WebRequest -Uri 'http://localhost:8080/openapi/v1.json' -OutFile "$env:USERPROFILE\Desktop\openapi-v1.json"
```

## 주의할 점
- 화면을 막아도 명세가 열려 있으면 API 목록 전체가 노출된다. 운영에서 명세까지 막을지는 그 명세를 읽어 가는 연동이 있는지 보고 정한다.
- 설정 키 오타는 기본값으로 처리되므로, 바꾼 뒤 화면·명세·실제 API 세 경로의 응답 코드를 모두 확인한다.
- Notepad++ 플러그인 설치는 인터넷에서 목록을 받으므로 외부망에서 한다.

## 참고
- OpenAPI Specification
- Scalar API Reference, Swagger UI 문서
- Notepad++ Plugins Admin
