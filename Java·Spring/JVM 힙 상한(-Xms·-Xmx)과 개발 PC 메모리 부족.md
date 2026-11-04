# JVM 힙 상한(-Xms·-Xmx)과 개발 PC 메모리 부족

> 분류: Java·Spring · 최근 갱신: 2026-11-03

## 한 줄 요약
IDE와 로컬 WAS를 함께 띄우는 개발 PC가 뜨거워지고 버벅이면, 힙 상한 없이 뜬 JVM들이 RAM을 나눠 먹다 GC를 계속 도는 경우가 많다. 각 JVM에 `-Xmx`를 정해 합계가 RAM 안에 들어오게 한다.

## 내용
### -Xms와 -Xmx
- `-Xms`: 시작할 때 잡는 힙 크기(s = start). 필수는 아니다.
- `-Xmx`: 힙이 커질 수 있는 최대치(x = max). `2048m`과 `2g`는 같다.
- `-Xmx`를 주지 않으면 JVM은 기본값으로 **물리 RAM의 1/4**까지 힙을 늘릴 수 있다. 16GB PC에서 JVM 하나가 약 4GB, 톰캣 두 개면 약 8GB까지 갈 수 있다.
- JVM은 상한까지 여유가 있으면 쓴 메모리를 OS에 잘 돌려주지 않는다. 그래서 오래 켜 둘수록 점유가 커진다.
- 상한을 너무 낮게 잡으면 `OutOfMemoryError: Java heap space`로 멈춘다. 그때 조금씩 올린다.

### 증상으로 알아보기
- 요청이 없는데도 서버가 응답하지 않거나(타임아웃) 500을 내고, CPU와 팬이 올라간다 → 힙이 차서 GC를 연달아 도는 상태일 가능성이 크다.
- Windows 이벤트 로그의 "가상 메모리 부족" 경고, 작업 폴더에 남은 `hs_err_pid*.log`(JVM 비정상 종료 기록)도 같은 신호다.
- Windows는 프로세스별 CPU 사용 기록을 남기지 않는다. 다시 켜기 전에 작업 관리자 "세부 정보" 탭을 CPU·메모리 순으로 정렬해 확인해야 원인을 잡을 수 있다.

### 어느 JVM의 옵션인지 구분하기
- IDE(IntelliJ 등) 자체의 힙은 IDE 설정 폴더의 `*.vmoptions` 파일(메뉴: 도움말 > 사용자 지정 VM 옵션 편집)에서 정한다. 로컬 WAS의 힙과 별개다.
- IDE에서 띄운 톰캣의 힙은 실행 구성(Run Configuration)의 VM options, 톰캣 `bin/setenv`, `JAVA_OPTS`/`CATALINA_OPTS` 환경 변수 중 하나에서 온다. 셋 다 비어 있으면 기본값(RAM 1/4)이다.
- 실제로 적용됐는지는 실행 중인 프로세스의 명령줄에 `-Xmx`가 들어 있는지로 확인한다. IDE는 옵션 파일을 내부에서 읽어 명령줄에 안 보일 수 있다. 정확한 값은 `jcmd <pid> VM.flags`로 본다(JDK에 포함, JRE만 있으면 없음).

### 합계로 계산하기
- 최악의 경우 = IDE 상한 + WAS 상한 × 대수 + DB 서버 + 브라우저·메신저. 이것이 RAM보다 크면 오래 켜 둘 때 페이징과 GC가 겹친다.
- 16GB 기준 예: IDE 4GB, 주 WAS 2GB, 보조 WAS 1GB 정도면 대부분의 로컬 개발에 충분하다. 안 쓰는 WAS는 꺼 둔다.

### 함께 알아 둘 것
- **Windows 빠른 시작**이 켜져 있으면 "시스템 종료"는 최대 절전처럼 동작해 커널·드라이버가 초기화되지 않는다. 커널 가동 시간이 며칠씩 이어진다. 완전히 초기화하려면 "다시 시작"을 쓴다.
- Chrome·Electron 기반 앱(데스크톱 메신저, AI 앱 등)은 화면(renderer)·GPU·utility·crashpad를 각각 프로세스로 띄운다. 작업 관리자에 프로세스가 10개 넘게 보여도 앱이 그만큼 켜진 것이 아니다. 메모리는 프로세스 합계로 본다.
- 일부 스토어 앱은 레지스트리 Run 목록이 아니라 앱 패키지의 "시작 작업"으로 자동 실행된다. 설정 > 앱 > 시작 프로그램에서 끄면 둘 다 처리된다.

## 핵심 코드
### 1) WAS 실행 구성의 VM options (IDE 실행 구성 또는 setenv)
```bash
# 주 WAS
-Xms512m -Xmx2048m -Dfile.encoding=UTF-8
# 보조 WAS
-Xms512m -Xmx1024m
```

### 2) 톰캣 setenv로 지정하는 경우 (bin 폴더에 새로 만듦)
```bat
rem bin\setenv.bat
set "CATALINA_OPTS=%CATALINA_OPTS% -Xms512m -Xmx2048m"
```

### 3) IDE 자체 힙 (예: idea64.exe.vmoptions)
```text
-Xms1024m
-Xmx4096m
```

### 4) 적용 확인 (PowerShell)
```powershell
# java 프로세스 명령줄에서 -Xmx 확인
Get-CimInstance Win32_Process -Filter "Name='java.exe'" |
  Select-Object ProcessId, @{n='Xmx';e={([regex]'-Xmx\S+').Match($_.CommandLine).Value}}

# 커널 가동 시간 (빠른 시작 때문에 종료해도 안 줄어드는지)
(Get-Date) - (Get-CimInstance Win32_OperatingSystem).LastBootUpTime
```

```bash
# JDK가 있으면 실제 적용된 힙 상한
jcmd <pid> VM.flags | findstr MaxHeapSize
```

## 주의할 점
- IDE가 열려 있는 동안 `*.vmoptions`를 바깥에서 고치면 IDE가 덮어쓸 수 있다. IDE 메뉴에서 바꾸고 재시작한다.
- 힙 상한은 힙만 제한한다. 메타스페이스·스레드 스택·다이렉트 버퍼 때문에 실제 프로세스 메모리는 `-Xmx`보다 수백 MB 더 크다.

## 참고
- Oracle Java 문서: java 명령 옵션(-Xms, -Xmx), jcmd
- Apache Tomcat 문서: RUNNING.txt (setenv, CATALINA_OPTS)
