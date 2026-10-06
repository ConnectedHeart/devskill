# 외부 API 판정 결과의 로컬 캐시 설계

> 분류: Java·Spring · 최근 갱신: 2026-10-06

## 한 줄 요약
화면마다 필요한 외부 API 판정(권한 여부 등)을 Caffeine 같은 JVM 로컬 캐시에 둘 때 정해야 하는 것: 성공 캐시, 실패 캐시, 요청 범위 재사용, 다중 서버에서의 한계.

## 내용
**계층을 나눠 생각한다**
- **request 속성** (`request.setAttribute`): 그 요청 하나가 처리되는 동안만 살아 있다. 한 요청 안에서 같은 외부 호출을 두 번 하지 않기 위한 용도다.
- **로컬 캐시**: 요청과 요청 사이에 결과를 유지한다. 서버 메모리에 있어 요청이 끝나도 남는다.

**실패 캐시(negative cache)가 필요한 이유**
- 실패는 보통 성공 캐시에 저장되지 않는다. 그러면 외부 서버가 죽어 있는 동안 화면을 열 때마다 타임아웃까지 기다리는 호출이 반복되고, 외부 서버 하나의 장애가 애플리케이션 전체의 화면 지연으로 번진다.
- 실패를 짧게(수십 초) 기억해 그 동안은 바로 기본값(예: 차단)으로 답하면 지연이 "수십 초에 한 번"으로 줄어든다.
- 대가: 일시적 오류 한 번에도 그 시간 동안 해당 사용자는 실패 상태로 보이고, 복구 반영이 그만큼 늦는다. 시간은 짧게 잡는다.

**수명(TTL) 정하기**
- 짧을수록 설정 변경이 빨리 반영되지만 외부 호출이 그만큼 늘고 캐시가 비어 느린 화면도 잦아진다. 수명을 1/5로 줄이면 호출은 최대 5배다.
- 로그인처럼 "새로 판정받아야 하는 시점"에 해당 사용자의 항목을 명시적으로 비우면 긴 수명의 단점을 줄일 수 있다.

**메모리 사용량**
- 응답 전체를 저장하지 말고 분기에 쓰는 값만 남긴다. 키 문자열 + boolean + 작은 맵이면 항목당 0.5~1KB 수준이라 수천 명이어도 수 MB다(객체 구조로 계산한 추정치).
- 캐시에는 로그인한 전원이 아니라 수명 안에 해당 화면을 연 사람만 들어 있다. 최대 건수를 정해 두면 넘칠 때 덜 쓰는 항목부터 버려지고, 버려진 항목은 다음에 한 번 더 호출될 뿐이다.

**다중 서버(로드밸런서)에서**
- 로컬 캐시는 서버마다 따로다. 외부 호출이 서버 대수만큼 늘고, 변경 반영 시점이 서버마다 달라 같은 사용자에게 결과가 잠깐 엇갈릴 수 있다. 명시적 비우기도 요청을 받은 서버에서만 일어난다.
- 그래도 어느 서버든 수명을 넘기지 않으므로 "최대 N분 지연"이라는 보장은 유지된다. 스티키 세션이면 차이가 거의 드러나지 않는다.
- 서버 간에 완전히 같은 결과가 필요하면 Redis 같은 외부 저장소나 DB로 옮겨야 한다.

**실패 시 기본값**
- 권한 판정처럼 "잘못 열리면 안 되는" 값은 실패 시 차단(fail-closed)으로 둔다. 단, 그 판정으로 관리 화면 진입까지 막으면 관리자가 스스로 잠겨 설정을 풀 수 없게 될 수 있으니 복구 경로를 따로 확인한다.
- 화면만 감추면 URL 직접 호출로 우회되므로, 같은 판정을 서버 쪽 엔드포인트에도 적용한다.

## 예시
```java
private final Cache<String, Permission> allowed = Caffeine.newBuilder()
        .expireAfterWrite(5, TimeUnit.MINUTES)
        .maximumSize(50_000)
        .build();

private final Cache<String, Boolean> failures = Caffeine.newBuilder()
        .expireAfterWrite(30, TimeUnit.SECONDS)
        .maximumSize(50_000)
        .build();

Permission resolve(String key) {
    Permission cached = allowed.getIfPresent(key);
    if (cached != null) return cached;
    if (failures.getIfPresent(key) != null) return Permission.DENIED;
    try {
        Permission p = callRemote();      // 연결·응답 타임아웃 필수
        allowed.put(key, p);
        return p;
    } catch (Exception e) {
        failures.put(key, Boolean.TRUE);
        return Permission.DENIED;
    }
}
```

## 주의할 점
- 캐시 키에 판정에 영향을 주는 값(사용자, 소속 등)을 모두 넣는다. 빠지면 소속을 바꿔도 이전 결과가 나온다.
- 한 요청에서 토큰 발급 같은 부수 효과가 있는 호출이 두 번 일어나지 않도록, 처음 결과를 request 속성에 담아 재사용한다.
- 메모리 추정치는 측정값이 아니다. 항목이 커질 수 있으면 실제로 재 본다.

## 참고
- Caffeine 위키 — Eviction, Population
