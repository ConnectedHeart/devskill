# Future로 외부 호출 전체 시간 제한하기

> 분류: Java·Spring · 최근 갱신: 2026-10-06

## 한 줄 요약
connect/read 타임아웃만으로는 호출 "전체 시간"이 제한되지 않으므로, 스레드풀에 작업을 맡기고 `Future.get(timeout)`으로 상한을 한 번 더 거는 패턴과 그 예외 처리 방법.

## 내용
**왜 한 번 더 감싸나**
- connect 타임아웃은 "연결이 맺어질 때까지", read 타임아웃은 "읽기 한 번이 끝날 때까지"의 제한이다. DNS 조회가 느리거나 응답이 조금씩 끊겨 오면 각 제한에는 안 걸리면서 전체 시간은 훨씬 길어질 수 있다.
- 로그인처럼 사용자가 기다리는 요청 안에서 외부 서버를 부를 때는 전체 시간에 상한을 두어야 요청 스레드가 오래 붙잡히지 않는다.

**흐름**
1. `executor.submit(callable)`로 실제 호출을 다른 스레드에 맡기고 `Future`를 받는다.
2. `future.get(timeout, unit)`로 그 시간만큼만 기다린다.

**예외 세 가지**

| 예외 | 언제 | 처리 |
|---|---|---|
| `TimeoutException` | 제한 시간 안에 안 끝남 | `future.cancel(true)` 후 호출부가 이해하는 예외로 변환 |
| `InterruptedException` | 기다리던 스레드 자체가 인터럽트됨 | `Thread.currentThread().interrupt()`로 플래그 복구 후 변환 |
| `ExecutionException` | 작업 스레드 안에서 예외 발생 | `getCause()`로 원래 예외를 꺼내 종류별로 다시 던짐 |

- 다른 스레드에서 난 예외는 `ExecutionException`으로 한 겹 싸여 온다. 벗기지 않으면 호출부가 "인증 거부"와 "통신 오류"를 구분할 수 없다.
- 변환을 한 곳에서 해 두면 호출부가 다룰 예외 종류가 두세 가지로 고정된다.

## 예시
```java
private TokenResponse requestToken(final Grant grant) throws IOException, AuthException {
    Future<TokenResponse> future = EXECUTOR.submit(() -> requestTokenBlocking(grant));
    try {
        return future.get(TIMEOUT_MILLIS, TimeUnit.MILLISECONDS);
    } catch (TimeoutException e) {
        future.cancel(true);
        throw new IOException("timeout", e);
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        throw new IOException("interrupted", e);
    } catch (ExecutionException e) {
        Throwable cause = e.getCause();
        if (cause instanceof AuthException) throw (AuthException) cause;
        if (cause instanceof IOException) throw (IOException) cause;
        throw new IOException(cause);
    }
}
```

## 주의할 점
- `future.cancel(true)`는 작업 스레드에 인터럽트를 걸 뿐이다. 블로킹 소켓 I/O 중인 스레드는 바로 멈추지 않고 소켓 자체 타임아웃이 끝날 때까지 스레드풀 자리를 차지한다. 그래서 안쪽 호출에도 connect/read 타임아웃을 반드시 건다.
- 스레드풀 크기가 작으면 외부 서버 장애 때 풀이 가득 차 이후 호출이 줄줄이 타임아웃된다. 풀 크기와 대기 큐를 장애 상황 기준으로 정한다.
- 화면(ajax) 타임아웃이 서버 쪽 최악 시간(각 단계 타임아웃의 합)보다 짧으면, 서버 응답 전에 화면이 먼저 일반 실패 메시지를 띄운다. 두 값을 같이 본다.

## 참고
- `java.util.concurrent.Future`, `ExecutorService` Javadoc
- Java Concurrency in Practice — 7장 Cancellation and Shutdown
