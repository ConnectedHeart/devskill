# nohup으로 jar 백그라운드 실행과 java 옵션 순서

> 분류: 기타 · 최근 갱신: 2026-10-15

## 한 줄 요약
`java` 명령에서 `-D`, `-cp` 같은 JVM 옵션은 `-jar 파일명` 앞에 와야 한다. 뒤에 쓰면 프로그램 인자로 넘어가 적용되지 않는다. `-jar`를 쓰면 `-cp`는 무시된다. 터미널을 닫아도 살아 있게 하려면 `nohup ... &`로 띄우되 출력을 어디로 보낼지 정한다.

## 내용
### 옵션 순서
- `java [JVM 옵션] -jar app.jar [프로그램 인자]` 구조다. `-jar app.jar` 뒤의 모든 토큰은 `main(String[] args)`로 들어간다.
- 잘못된 예: `java -jar app.jar -cp $CLASSPATH -Djava.library.path=$LD_LIBRARY_PATH` → `java.library.path`가 설정되지 않아 네이티브 라이브러리(`.so`)를 못 찾는다(`UnsatisfiedLinkError`).
- `-jar`로 실행하면 클래스패스는 jar 매니페스트의 `Class-Path`로 정해지고 `-cp`는 무시된다. 외부 jar를 같이 써야 하면 매니페스트에 넣거나 `-cp a.jar:b.jar 메인클래스` 방식으로 실행한다.
- 네이티브 라이브러리를 시스템 lib 경로(`/usr/lib64` 등)에 넣었다면 `-Djava.library.path`는 생략할 수 있다. `LD_LIBRARY_PATH`는 JVM이 시작할 때 `java.library.path` 기본값에 반영된다.

### nohup과 출력
- `nohup 명령 &`는 터미널 연결이 끊겨도(HUP 시그널) 프로세스를 유지하고 백그라운드로 보낸다.
- 출력을 지정하지 않으면 현재 폴더에 `nohup.out`이 생기고 계속 커진다.
- `> /dev/null 2>&1`로 버리면 파일은 안 생기지만 기동 실패 로그도 남지 않는다. 애플리케이션이 자체 로그 파일을 쓰는지 먼저 확인하고, 아니면 `> app.out 2>&1`처럼 파일로 보낸다.
- 서비스로 오래 돌릴 것이라면 systemd 유닛이 재시작·로그 관리 면에서 낫다.

## 예시
```bash
# 환경 변수 (예: 셸 프로필)
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/opt/app/lib64

# 올바른 실행
nohup java -Djava.library.path=$LD_LIBRARY_PATH -jar app-0.0.1-SNAPSHOT.jar > /opt/app/app.out 2>&1 &

# 기동 확인
tail -f /opt/app/app.out
ps -ef | grep app-0.0.1
```

## 주의할 점
- 가이드 문서에 적힌 명령을 그대로 옮기기 전에 옵션 순서를 본다. 문서마다 `-cp`·`-D` 위치가 다르게 적혀 있는 경우가 흔하다.
- `&`만 붙이고 `nohup`을 빼면 SSH 세션이 끊길 때 프로세스가 같이 죽을 수 있다(셸 설정에 따라 다름).

## 참고
- `java` 명령 문서: Standard Options (-cp, -D, -jar)
- GNU coreutils: nohup
