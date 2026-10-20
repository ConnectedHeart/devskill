# 클로드 지식 내용

Claude와 작업하면서 알게 된 기술 지식을 분류별·주제별로 정리한 노트입니다. 특정 회사나 프로젝트에 한정된 내용은 빼고, 다른 곳에서도 쓸 수 있는 내용만 담습니다.

- 구조: `<분류>/<주제명>.md`
- 같은 주제는 한 파일에 계속 보탭니다.

## Git
- [git bundle 생성과 병합](Git/git bundle 생성과 병합.md) — 저장소를 파일 하나로 옮기는 방법, FETCH_HEAD로 병합하는 이유, --no-ff
- [cherry-pick 충돌 해결과 검증](Git/cherry-pick 충돌 해결과 검증.md) — 선행 커밋 찾기와 순서, 충돌이 많은 이유 3가지, worktree 시험, 공백만 다른 충돌, 검증 절차
- [병합 커밋에서 사라진 코드 찾기](Git/병합 커밋에서 사라진 코드 찾기.md) — 같은 부모를 재병합해 비교, 지워짐 판정 기준, 복구와 이력 맞추기
- [git commit --fixup과 autosquash](Git/git commit --fixup과 autosquash.md) — 뒤에 커밋이 쌓인 뒤 예전 커밋 고치기, amend와의 차이, 푸시 전에만 쓰는 이유
- [푸시된 브랜치에서 커밋 빼서 다른 브랜치로 옮기기](Git/푸시된 브랜치에서 커밋 빼서 다른 브랜치로 옮기기.md) — force-push 대신 revert, 빠지는 정의의 호출처 확인, worktree 작업과 긴 경로 오류
- [format-patch와 am으로 커밋 옮기기와 적용 전 검증](Git/format-patch와 am으로 커밋 옮기기와 적용 전 검증.md) — --cherry-pick으로 안 넘어간 커밋 찾기, 패치 생성, 임시 인덱스로 apply --cached --check, 여러 패치 순차 점검, 공백 경고

## 인증·보안
- [Keycloak audience(aud)와 azp 검증](인증·보안/Keycloak audience(aud)와 azp 검증.md) — aud·azp의 의미, Audience 매퍼 추가, API 서버 검증과 적용 순서
- [1회용 코드 SSO와 refresh token 로테이션](인증·보안/1회용 코드 SSO와 refresh token 로테이션.md) — 토큰을 브라우저로 내보내지 않는 SSO 흐름, 오류 위치 구분, 로테이션 충돌
- [Keycloak 토큰 클레임(jti·sid·sub)과 로그아웃 범위](인증·보안/Keycloak 토큰 클레임(jti·sid·sub)과 로그아웃 범위.md) — 재발급 시 바뀌는 값, 세션 단위 로그아웃, 자체 검증 서버의 틈
- [토큰 보관 위치와 재발급 기준](인증·보안/토큰 보관 위치와 재발급 기준.md) — 메모리·쿠키·세션·DB 비교, 수명 절반 기준, 토큰 응답 캐시 금지, 쿠키 공유 범위
- [Bearer 토큰과 세션 ID를 API 인증에 쓸 때의 차이](인증·보안/Bearer 토큰과 세션 ID를 API 인증에 쓸 때의 차이.md) — Authorization 헤더, invalid_grant 구분, 세션 ID 유출 범위, 전용 불투명 토큰
- [서버 간 호출 401 원인 좁히기와 로그 설계](인증·보안/서버 간 호출 401 원인 좁히기와 로그 설계.md) — 되묻는 호출 유무로 거부 지점 가르기, trace id, 비밀 값 로그 금지, GUID와 UUID
- [JWT 구조와 서명 검증 조건](인증·보안/JWT 구조와 서명 검증 조건.md) — 암호화가 아닌 서명, 디코딩 방법, iss·exp·aud·JWKS 검증과 401 점검 순서
- [LDAP bind로 AD 비밀번호 확인하기](인증·보안/LDAP bind로 AD 비밀번호 확인하기.md) — simple bind 판정, 비밀번호 동기화 패턴, 빈 비밀번호·인증서 검증·장애 처리 주의
- [비밀번호로 토큰을 발급하는 API를 추가할 때 점검할 것](인증·보안/비밀번호로 토큰을 발급하는 API를 추가할 때 점검할 것.md) — 세션 기반 토큰 수명, 옮길 보안 검사, 실패 로그, 평문 전송과 쿼리스트링 금지

## Java·Spring
- [synchronized와 잠금 객체 나누기(lock striping)](Java·Spring/synchronized와 잠금 객체 나누기(lock striping).md) — 잠금 객체의 의미, 키 해시로 N개 잠금 고르기, JVM 한계
- [Spring 트랜잭션 전파와 자동 커밋](Java·Spring/Spring 트랜잭션 전파와 자동 커밋.md) — 트랜잭션 매니저와 커넥션 풀의 역할, REQUIRES_NEW가 필요한 경우, 정리 작업을 트랜잭션 밖으로
- [외부 API 판정 결과의 로컬 캐시 설계](Java·Spring/외부 API 판정 결과의 로컬 캐시 설계.md) — 성공·실패 캐시, request 속성과의 역할 구분, 다중 서버 한계
- [Future로 외부 호출 전체 시간 제한하기](Java·Spring/Future로 외부 호출 전체 시간 제한하기.md) — connect/read 타임아웃의 빈틈, ExecutionException 벗기기, cancel의 한계
- [RestTemplate JSON 호출의 숫자 타입과 타임아웃](Java·Spring/RestTemplate JSON 호출의 숫자 타입과 타임아웃.md) — Gson·json-simple·Jackson의 타입 차이, 전용 RestTemplate
- [코드 점검에서 자주 나오는 결함 패턴](Java·Spring/코드 점검에서 자주 나오는 결함 패턴.md) — 순회 중 삭제, 예외 삼킴, 확장자 판별, 프록시를 거치지 않는 내부 호출, 문자열 == 비교, null 확인 순서, 단위 계산 인덱스, 검증 뒤 return 누락
- [RequestBody와 요청 파라미터(getParameterNames)의 차이](Java·Spring/RequestBody와 요청 파라미터(getParameterNames)의 차이.md) — JSON 본문과 쿼리스트링·폼의 구분, 엄격한 파라미터 인터셉터, 쿼리스트링 비밀번호 로그
- [의존성 제거 후 다른 라이브러리 클래스 import로 깨지는 컴파일](Java·Spring/의존성 제거 후 다른 라이브러리 클래스 import로 깨지는 컴파일.md) — Tomcat Base64 import, Commons Codec·java.util.Base64로 교체
- [iBATIS SqlMap 사용 시 주의점](Java·Spring/iBATIS SqlMap 사용 시 주의점.md) — resultClass 없는 select는 null, Map 키 대소문자, CDATA 안 주석, isNotEmpty 기본값, IN 절 1000개 분할
- [요청 중 만든 임시 파일 정리](Java·Spring/요청 중 만든 임시 파일 정리.md) — finally에서 삭제, 확장자가 아닌 플래그로 판단, 스케줄러 정리 대상 확인, 동시 요청 파일명
- [서블릿 필터 URL 패턴과 한글 파일명 깨짐](Java·Spring/서블릿 필터 URL 패턴과 한글 파일명 깨짐.md) — *.do에만 걸린 인코딩 필터, 확장자 없는 주소의 multipart 깨짐, {id:.+} 경로 변수가 확장자를 먹는 문제

## DB·SQL
- [AUTO_INCREMENT가 건너뛰며 증가하는 이유](DB·SQL/AUTO_INCREMENT가 건너뛰며 증가하는 이유.md) — auto_increment_increment와 Galera, LAST_INSERT_ID와 MAX의 차이
- [SELECT FOR UPDATE로 다중 서버 중복 갱신 막기](DB·SQL/SELECT FOR UPDATE로 다중 서버 중복 갱신 막기.md) — 행 잠금 후 재판정, JVM 잠금과 함께 쓰는 이유
- [시각 컬럼 타입 선택(epoch millis와 날짜 타입)](DB·SQL/시각 컬럼 타입 선택(epoch millis와 날짜 타입).md) — 타임존 위험, UTC 기록 컬럼, MySQL·Oracle 컬럼 타입 변경 차이
- [PostgreSQL 개요와 MySQL·Oracle 문법 차이](DB·SQL/PostgreSQL 개요와 MySQL·Oracle 문법 차이.md) — 특징, 문법 비교표, 식별자 소문자화와 엄격한 형변환
- [DB별 SQL에서 조용히 틀리는 조건식 패턴](DB·SQL/DB별 SQL에서 조용히 틀리는 조건식 패턴.md) — Oracle || 와 - 우선순위, MySQL IN 안의 OR, BETWEEN 인자 순서, 날짜에 숫자 빼기, 시각 붙은 날짜 = 비교, ORDER BY 없는 순서 의존

## 네트워크
- [LDAP 연결 끊김 원인 구분](네트워크/LDAP 연결 끊김 원인 구분.md) — 오류 메시지별 의미, TTL 비교로 끊은 주체 찾기
- [SSH 공개키 인증과 비대화형 실행 옵션](네트워크/SSH 공개키 인증과 비대화형 실행 옵션.md) — ssh-copy-id와 Windows 대체 명령, BatchMode 등 -o 옵션, 종료 코드 255와 오류 구분
- [WAF 웹 애플리케이션 방화벽과 오탐 대응](네트워크/WAF 웹 애플리케이션 방화벽과 오탐 대응.md) — L7 검사와 룰셋, 정상 입력 403 오탐 증상, 서버 로그에 요청이 없으면 의심, 예외 등록·인코딩
- [HTTP Range 요청과 206 응답(미디어 재생 프록시)](네트워크/HTTP Range 요청과 206 응답(미디어 재생 프록시).md) — audio 태그의 구간 요청, iOS Safari 206 필수, 2단 프록시에서 상태코드·Content-Range 전달, RestTemplate 대신 직접 스트림 중계, 탭 즉시 재생
- [SSH config Host 별칭과 hosts 파일의 차이](네트워크/SSH config Host 별칭과 hosts 파일의 차이.md) — git 원격의 짧은 호스트명 정체, SSH만 해석되어 ping은 안 됨, ssh -G 확인
- [WebSocket 종료 코드로 끊긴 구간 찾기](네트워크/WebSocket 종료 코드로 끊긴 구간 찾기.md) — RFC 6455 종료 코드, 다단 중계에서 코드·사유 설계, 1회용 티켓, 프록시 Upgrade·유휴 타임아웃

## Windows·자동화
- [작업 스케줄러로 스크립트 자동 실행](Windows·자동화/작업 스케줄러로 스크립트 자동 실행.md) — UAC 없이 관리자 스크립트 실행, idle 감지, 창 없는 실행
- [예약 재시작과 전원 설정](Windows·자동화/예약 재시작과 전원 설정.md) — shutdown /g와 ARSO, 절전 방지 powercfg
- [PowerShell과 Git Bash 사용 시 주의점](Windows·자동화/PowerShell과 Git Bash 사용 시 주의점.md) — curl 별칭, CRLF 변환, 스크립트 인코딩, 1건 배열
- [PowerShell 기초 문법(Java 개발자용)](Windows·자동화/PowerShell 기초 문법(Java 개발자용).md) — 변수 스코프, 단어 연산자, @ 기호, 객체 파이프, cmdlet 호출, 경로 명령
- [PowerShell GUI 도구에서 외부 프로세스 출력 실시간 표시](Windows·자동화/PowerShell GUI 도구에서 외부 프로세스 출력 실시간 표시.md) — 창 없는 실행과 출력 리다이렉트, DoEvents, 프로세스 트리 종료, exe로 감싸도 소스가 보이는 이유

## Claude Code
- [Remote Control로 휴대폰에서 이어 쓰기](Claude Code/Remote Control로 휴대폰에서 이어 쓰기.md) — 동작 구조, 켜는 방법, 끊기는 조건, 다른 기기에 세션 제목이 보이는 이유
- [하네스 엔지니어링 개념](Claude Code/하네스 엔지니어링 개념.md) — 모델을 뺀 나머지 전부, 에이전트 루프·도구·컨텍스트·메모리·가드레일·피드백 루프, 지침 파일과 스킬 나누기
- [훅(hook) 작성 요령](Claude Code/훅(hook) 작성 요령.md) — 빠르게 끝내는 방법, 훅 출력이 컨텍스트로 들어가는 점
- [예약 작업 무인 실행과 권한 규칙](Claude Code/예약 작업 무인 실행과 권한 규칙.md) — 고정 스크립트와 좁은 허용 규칙, 놓친 날 보충 장치의 구멍, 권한 요청 대기로 멈춘 실행, 목차 기반 요약
- [로컬 환경 전환 불가 메시지](Claude Code/로컬 환경 전환 불가 메시지.md) — 메시지가 나오는 경우, + 버튼 세션은 실제로 로컬 실행, 재시작·폴더 재지정·정책 확인 순서
- [Agent Skills SKILL.md 위치(Claude Code와 Codex)](Claude Code/Agent Skills SKILL.md 위치(Claude Code와 Codex).md) — SKILL.md 표준, .claude/skills와 .agents/skills 차이, AGENTS.md에 핵심 규칙 두기, 규칙·절차 분리 기준, CLAUDE.md import, 폴더 링크

## 기타
- [JavaScript var 스코프와 중첩 루프 버그](기타/JavaScript var 스코프와 중첩 루프 버그.md) — 함수 스코프로 루프 변수가 공유되는 원인과 증상, 수정 방법
- [Docker 컨테이너 파일을 호스트로 꺼내기](기타/Docker 컨테이너 파일을 호스트로 꺼내기.md) — docker cp 후 SFTP로 받기, 볼륨 마운트 경로 확인
- [앱·사이트 사용 제한 프로그램의 구현 방식과 한계](기타/앱·사이트 사용 제한 프로그램의 구현 방식과 한계.md) — Windows 프로세스 감시, 설정 파일 잠금·레지스트리 사본으로 초기화 우회 막기, Android 접근성 서비스, 기본 제공 기능과 비교
- [iframe src 변경과 브라우저 방문 기록](기타/iframe src 변경과 브라우저 방문 기록.md) — 뒤로가기를 여러 번 눌러야 하는 원인, location.replace, history.length로 추적
- [외부 검색엔진 연동과 검색 결과 권한 필터링](기타/외부 검색엔진 연동과 검색 결과 권한 필터링.md) — 색인용 뷰, 트리거 증분 색인, 권한 색인과 접근 식별자 목록 대조
- [톰캣 폴더 교체 배포 스크립트 설계](기타/톰캣 폴더 교체 배포 스크립트 설계.md) — 준비·교체 분리, 준비 완료 표시 파일, mkdir 잠금, ps -o 함정, 기동 확인과 되돌리기
- [목록 화면에서 선택 행과 전역 변수가 어긋나는 버그](기타/목록 화면에서 선택 행과 전역 변수가 어긋나는 버그.md) — 체크박스·Ctrl 클릭이 변수를 갱신하지 않아 다른 항목이 처리되는 원인, 수정과 재현 요령
- [외부 응답을 화면에 넣는 방식(텍스트 노드와 innerHTML)](기타/외부 응답을 화면에 넣는 방식(텍스트 노드와 innerHTML).md) — 태그가 글자로 보이는 이유, 컨트롤 노출, data-* 속성과 경로별 필터
- [nohup으로 jar 백그라운드 실행과 java 옵션 순서](기타/nohup으로 jar 백그라운드 실행과 java 옵션 순서.md) — -D·-cp는 -jar 앞에, -jar이면 -cp 무시, nohup 출력 처리와 기동 로그
- [HWP·PDF 문서에서 텍스트 뽑기](기타/HWP·PDF 문서에서 텍스트 뽑기.md) — hwplib·pyhwp, pdftotext의 한글 CID 글꼴(poppler-data) 누락, 추출 결과의 한계
- [날짜 기준 전송 배치의 재전송 설계](기타/날짜 기준 전송 배치의 재전송 설계.md) — 이력 날짜의 의미 맞추기, 행 없으면 생성, 서버 검증, 전송 플래그 되돌리기, 날짜 계산 한 곳에서, 갱신 키 유지
- [파일 경로만 넘기는 서버 간 연동과 공유 저장소](기타/파일 경로만 넘기는 서버 간 연동과 공유 저장소.md) — 공유가 깨지면 200 + 빈 결과, 마운트·기준 경로·권한 확인, 로컬 개발 서버의 구조적 한계
- [ps STAT 값(Ss+)과 남아 있는 tail -F 프로세스](기타/ps STAT 값(Ss+)과 남아 있는 tail -F 프로세스.md) — STAT 문자 의미, SSH로 띄운 tail이 세션 리더가 되는 이유, grep에 걸리는 로그 경로
- [브라우저 마이크 녹음 기능을 붙일 때 확인할 것](기타/브라우저 마이크 녹음 기능을 붙일 때 확인할 것.md) — https 보안 컨텍스트, 앱 웹뷰 권한, 화면 이탈 시 마이크 해제, 다단 WebSocket 중계와 단기 티켓, 프록시 Upgrade·고정 세션
- [HTML select 펼친 목록에 CSS가 안 먹는 이유](기타/HTML select 펼친 목록에 CSS가 안 먹는 이유.md) — 네이티브 옵션 목록, jQuery Mobile 자체 목록, 둥근 팝업에서 글자 잘림
