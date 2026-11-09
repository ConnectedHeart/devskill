# 클로드 지식 내용

Claude와 작업하면서 알게 된 기술 지식을 분류별·주제별로 정리한 노트입니다. 특정 회사나 프로젝트에 한정된 내용은 빼고, 다른 곳에서도 쓸 수 있는 내용만 담습니다.

- 구조: `<대분류>/<주제명>.md` 또는 `<대분류>/<중분류>/<주제명>.md`. 중분류는 한 대분류 안에서 좁은 주제의 노트가 여럿일 때만 만든다 (예: `네트워크/웹소켓/`). 목차에서는 `## 대분류` 아래 `### 중분류`로 묶는다.
- 같은 주제는 한 파일에 계속 보탭니다.

## Git
- [git bundle 생성과 병합](Git/git bundle 생성과 병합.md) — 저장소를 파일 하나로 옮기는 방법, FETCH_HEAD로 병합하는 이유, --no-ff
- [cherry-pick 충돌 해결과 검증](Git/cherry-pick 충돌 해결과 검증.md) — 선행 커밋 찾기와 순서, 충돌이 많은 이유 3가지, 선행 커밋 필요와 옆줄 겹침 구분, worktree·merge-tree 시험, 연속 작업 묶음 중간 시점 결함, push 전 rebase --onto로 하나 빼기, 공백만 다른 충돌, 검증 절차
- [git stash 적용 충돌 알아보기와 오래된 stash 정리](Git/git stash 적용 충돌 알아보기와 오래된 stash 정리.md) — Stashed changes 표시와 MERGE_HEAD 부재, reflog로 stash 경위 추적, HEAD 쪽 해결, 보관 후 drop
- [브랜치 간 미반영 커밋 찾기(작성자·커미터·patch-id)](Git/브랜치 간 미반영 커밋 찾기(작성자·커미터·patch-id).md) — 작성일·커밋일과 --since, 작성자·커미터 차이, 여러 표기 매핑, 역병합 구조에서 --cherry-pick이 짝을 못 찾는 이유
- [병합 커밋에서 사라진 코드 찾기](Git/병합 커밋에서 사라진 코드 찾기.md) — 같은 부모를 재병합해 비교, 지워짐 판정 기준, 복구와 이력 맞추기
- [git commit --fixup과 autosquash](Git/git commit --fixup과 autosquash.md) — 뒤에 커밋이 쌓인 뒤 예전 커밋 고치기, amend와의 차이, 푸시 전에만 쓰는 이유
- [푸시된 브랜치에서 커밋 빼서 다른 브랜치로 옮기기](Git/푸시된 브랜치에서 커밋 빼서 다른 브랜치로 옮기기.md) — force-push 대신 revert, 최신 순 revert, 함께 빼야 할 커밋·다른 저장소 영향, 빠지는 정의의 호출처 확인, worktree 작업과 긴 경로 오류
- [format-patch와 am으로 커밋 옮기기와 적용 전 검증](Git/format-patch와 am으로 커밋 옮기기와 적용 전 검증.md) — git am·apply·cherry-pick 비교, --cherry-pick으로 안 넘어간 커밋 찾기, 패치 생성, am --keep-non-patch로 [태그] 유지, 커밋 제목 파일명, 선행 패치 순서, 임시 인덱스로 apply --cached --check, 일부 패치 제외 적용, 멈췄을 때 abort·3way·skip, -3 원리와 fake ancestor, 멈춘 상태 읽기, conflictStyle diff3, 적용 스크립트 사전 검사, PowerShell 글롭 미전개, 빠진 패치 번호, stash 후 적용
- [git 한글 경로 깨짐과 스테이징 후 삭제된 파일](Git/git 한글 경로 깨짐과 스테이징 후 삭제된 파일.md) — core.quotepath, 터미널 인코딩, 인덱스에서 되살리기와 rm --cached, restore --staged
- [Windows PC에 Gitea로 git 서버 띄우기](Git/Windows PC에 Gitea로 git 서버 띄우기.md) — 공유 방식 비교, 위험한 기본값 막기, ROOT_URL과 이름 해석, 방화벽 허용 규칙을 먼저 넣는 이유(취소=차단 규칙), 팀·협업자 권한, safe.bareRepository 상속으로 push 404, 첫 clone OAuth 멈춤
- [git 병합(3-way merge)과 리버트·체리픽이 섞일 때 결과](Git/git 병합(3-way merge)과 리버트·체리픽이 섞일 때 결과.md) — 공통 조상·양 끝 비교 규칙표, 넣고 뺀 변경의 상쇄, 리버트가 반대 브랜치로 넘어가는 조건, 리버트된 커밋 체리픽 실수, 작업트리에 잡힌 브랜치

## 인증·보안
- [Keycloak audience(aud)와 azp 검증](인증·보안/Keycloak audience(aud)와 azp 검증.md) — aud·azp의 의미, Audience 매퍼 추가, API 서버 검증과 적용 순서
- [1회용 코드 SSO와 refresh token 로테이션](인증·보안/1회용 코드 SSO와 refresh token 로테이션.md) — 토큰을 브라우저로 내보내지 않는 SSO 흐름, 오류 위치 구분, 기존 로그인 세션 대신 전용 세션 발급, 로테이션 충돌
- [Keycloak 토큰 클레임(jti·sid·sub)과 로그아웃 범위](인증·보안/Keycloak 토큰 클레임(jti·sid·sub)과 로그아웃 범위.md) — 재발급 시 바뀌는 값, 세션 단위 로그아웃, 자체 검증 서버의 틈, 앱 로그아웃에 OIDC 로그아웃 붙이기와 응답별 처리
- [토큰 보관 위치와 재발급 기준](인증·보안/토큰 보관 위치와 재발급 기준.md) — 메모리·쿠키·세션·DB 비교, 수명 절반 기준, 토큰 응답 캐시 금지, 쿠키 공유 범위
- [Bearer 토큰과 세션 ID를 API 인증에 쓸 때의 차이](인증·보안/Bearer 토큰과 세션 ID를 API 인증에 쓸 때의 차이.md) — Authorization 헤더, invalid_grant 구분, 세션 ID 유출 범위, 전용 불투명 토큰
- [서버 간 호출 401 원인 좁히기와 로그 설계](인증·보안/서버 간 호출 401 원인 좁히기와 로그 설계.md) — 되묻는 호출 유무로 거부 지점 가르기, 401과 403 구분, 헤더 이름 목록으로 위치 불일치 찾기, access 로그에 없으면 WAS 미도달, trace id, 비밀 값 로그 금지, GUID와 UUID
- [JWT 구조와 서명 검증 조건](인증·보안/JWT 구조와 서명 검증 조건.md) — 암호화가 아닌 서명, 디코딩 방법, iss·exp·aud·JWKS 검증과 401 점검 순서
- [LDAP bind로 AD 비밀번호 확인하기](인증·보안/LDAP bind로 AD 비밀번호 확인하기.md) — simple bind 판정, 비밀번호 동기화 패턴, 빈 비밀번호·인증서 검증·장애 처리 주의
- [비밀번호로 토큰을 발급하는 API를 추가할 때 점검할 것](인증·보안/비밀번호로 토큰을 발급하는 API를 추가할 때 점검할 것.md) — 세션 기반 토큰 수명, 옮길 보안 검사, 실패 로그, 평문 전송과 쿼리스트링 금지
- [클라이언트가 만들 수 있는 값으로 권한을 판정할 때의 구멍](인증·보안/클라이언트가 만들 수 있는 값으로 권한을 판정할 때의 구멍.md) — 난독화 토큰 접두어 위조, X-Forwarded-For 조작과 구분자 주입, 서버 저장 속성·서명·접속 IP 비교, 예외 계정 허용 시 점검
- [화면 제한과 서버 측 접근 확인(IDOR)](인증·보안/화면 제한과 서버 측 접근 확인(IDOR).md) — 화면 숨김은 보안이 아님, ID로 데이터를 꺼내는 API만 확인 필요, 저장 전 대상의 소유 판정, 클라이언트가 보낸 값 신뢰 금지, 권한은 ID로·파일은 경로로 고르는 다운로드와 ID+순번 방식, 흩어진 판정의 구멍(1단계만 판정·통과 플래그·못 찾으면 통과·경로별 조합·관리자 API)과 판정 통일·전수조사 요령
- [URL의 ID 파라미터 숨기기와 서명 링크 토큰](인증·보안/URL의 ID 파라미터 숨기기와 서명 링크 토큰.md) — 사용자 본인에게는 숨길 수 없는 이유, 목표별 방법(POST·replaceState·토큰), DB 저장 토큰과 HMAC 서명 토큰 비교, 새로고침·메일 링크·후속 요청 한계
- [인증 실패 응답 문구와 거부 사유 로그](인증·보안/인증 실패 응답 문구와 거부 사유 로그.md) — 판정 기준·설정 상태를 흘리는 문구, code 유지·message만 일반화, request 속성으로 사유 전달, 값 없는 사유 로그와 로그 위조 방지

## Java·Spring
- [synchronized와 잠금 객체 나누기(lock striping)](Java·Spring/synchronized와 잠금 객체 나누기(lock striping).md) — 잠금 객체의 의미, 키 해시로 N개 잠금 고르기, JVM 한계
- [Spring 트랜잭션 전파와 자동 커밋](Java·Spring/Spring 트랜잭션 전파와 자동 커밋.md) — 트랜잭션 매니저와 커넥션 풀의 역할, REQUIRES_NEW가 필요한 경우, 정리 작업을 트랜잭션 밖으로
- [외부 API 판정 결과의 로컬 캐시 설계](Java·Spring/외부 API 판정 결과의 로컬 캐시 설계.md) — 성공·실패 캐시, request 속성과의 역할 구분, 다중 서버 한계
- [Future로 외부 호출 전체 시간 제한하기](Java·Spring/Future로 외부 호출 전체 시간 제한하기.md) — connect/read 타임아웃의 빈틈, ExecutionException 벗기기, cancel의 한계
- [RestTemplate JSON 호출의 숫자 타입과 타임아웃](Java·Spring/RestTemplate JSON 호출의 숫자 타입과 타임아웃.md) — Gson·json-simple·Jackson의 타입 차이, 전용 RestTemplate
- [코드 점검에서 자주 나오는 결함 패턴](Java·Spring/코드 점검에서 자주 나오는 결함 패턴.md) — 순회 중 삭제, 예외 삼킴, 확장자 판별, 프록시를 거치지 않는 내부 호출, 문자열 == 비교, null 확인 순서, 단위 계산 인덱스, 루프 안 파라미터 재대입과 초기값 없는 선언, 래퍼의 인자 전달 누락, 기본값 if 블록에 갇힌 검사(괄호 위치), 검증 뒤 return 누락
- [RequestBody와 요청 파라미터(getParameterNames)의 차이](Java·Spring/RequestBody와 요청 파라미터(getParameterNames)의 차이.md) — JSON 본문과 쿼리스트링·폼의 구분, 엄격한 파라미터 인터셉터, 쿼리스트링 비밀번호 로그
- [의존성 제거 후 다른 라이브러리 클래스 import로 깨지는 컴파일](Java·Spring/의존성 제거 후 다른 라이브러리 클래스 import로 깨지는 컴파일.md) — Tomcat Base64 import, Commons Codec·java.util.Base64로 교체
- [iBATIS SqlMap 사용 시 주의점](Java·Spring/iBATIS SqlMap 사용 시 주의점.md) — resultClass 없는 select는 null, Map 키 대소문자와 DB별 매퍼 한쪽 누락·기계적 대조, 들어올 수 없는 값과 비교해 죽은 isEqual 블록, CDATA 안 주석, isNotEmpty 기본값, IN 절 1000개 분할
- [목록 행마다 서비스를 부르는 N+1과 쪽 단위 일괄 조회](Java·Spring/목록 행마다 서비스를 부르는 N+1과 쪽 단위 일괄 조회.md) — 행 수에 비례하는 시간으로 알아보기, 트랜잭션 AOP·커넥션 검증 왕복, ID 모아 IN 조회 후 Map, 판정 조건·기본값 유지, 칸마다 같은 설정값 읽기, 트리 노드별 확인 요청을 일괄 요청으로, 일괄 조회가 기대는 인덱스와 패치 누락
- [JVM 힙 상한(-Xms·-Xmx)과 개발 PC 메모리 부족](Java·Spring/JVM 힙 상한(-Xms·-Xmx)과 개발 PC 메모리 부족.md) — 기본 힙 RAM 1/4, GC 폭주 증상, IDE와 WAS 옵션 구분·적용 확인, 합계 계산, 빠른 시작과 커널 가동 시간, Electron 다중 프로세스, Maven surefire forkCount=1C로 테스트 JVM 메모리 부족
- [요청 중 만든 임시 파일 정리](Java·Spring/요청 중 만든 임시 파일 정리.md) — finally에서 삭제, 확장자가 아닌 플래그로 판단, 스케줄러 정리 대상 확인, 동시 요청 파일명
- [파일 API를 S3 오브젝트 스토리지로 감쌀 때 달라지는 동작](Java·Spring/파일 API를 S3 오브젝트 스토리지로 감쌀 때 달라지는 동작.md) — 폴더 없는 키 구조, exists·listFiles prefix·renameTo 비원자성, new File·경로 전달 연동이 깨지는 이유, 범위 GET 스트리밍, 리뷰용 grep
- [서블릿 필터 URL 패턴과 한글 파일명 깨짐](Java·Spring/서블릿 필터 URL 패턴과 한글 파일명 깨짐.md) — *.do에만 걸린 인코딩 필터, 확장자 없는 주소의 multipart 깨짐, {id:.+} 경로 변수가 확장자를 먹는 문제
- [Spring에 선택적 두 번째 데이터소스 추가하기](Java·Spring/Spring에 선택적 두 번째 데이터소스 추가하기.md) — JNDI와 properties 직접 연결, 빈 기본값·lazy-init·autowire-candidate로 기동 유지, URL 없으면 기존 경로, DB 종류 메타데이터 판별, 트랜잭션 밖 직접 JDBC
- [HttpClient 커넥션 풀 크기와 자기 서버 호출 교착](Java·Spring/HttpClient 커넥션 풀 크기와 자기 서버 호출 교착.md) — 생성 방식별 기본 풀(5·2·무제한), 싱글톤화로 생기는 병목, 동시 호출 수로 크기 정하기, maxThreads 40% 이내 이유, 풀 대기 상한, 공유 쿠키 저장소·시스템 프로퍼티 누락, 같은 풀 다른 타임아웃, 팩토리 값이 설정 키를 이기는 실제 타임아웃, 긴 기본 읽기 타임아웃을 물려받는 외부 호출, 남은 new RestTemplate 찾기, 쿠키 미저장 뒤 상대 서버 세션 증가
- [반복 일정 전개 조회에서 기간 밖 반복 건너뛰기](Java·Spring/반복 일정 전개 조회에서 기간 밖 반복 건너뛰기.md) — 기간과 무관하게 펼치는 구조의 N+1, 종료 방식별 건너뛰기 조건과 근거, 메소드 지역 메모, 응답 바이트 비교·번갈아 측정·Com_select로 쿼리 수 세기

## DB·SQL
- [AUTO_INCREMENT가 건너뛰며 증가하는 이유](DB·SQL/AUTO_INCREMENT가 건너뛰며 증가하는 이유.md) — auto_increment_increment와 Galera, LAST_INSERT_ID와 MAX의 차이, DDL에 따라온 덤프 시작값
- [SELECT FOR UPDATE로 다중 서버 중복 갱신 막기](DB·SQL/SELECT FOR UPDATE로 다중 서버 중복 갱신 막기.md) — 행 잠금 후 재판정, JVM 잠금과 함께 쓰는 이유
- [시각 컬럼 타입 선택(epoch millis와 날짜 타입)](DB·SQL/시각 컬럼 타입 선택(epoch millis와 날짜 타입).md) — 타임존 위험, UTC 기록 컬럼, MySQL·Oracle 컬럼 타입 변경 차이
- [PostgreSQL 개요와 MySQL·Oracle 문법 차이](DB·SQL/PostgreSQL 개요와 MySQL·Oracle 문법 차이.md) — 특징, 문법 비교표, 식별자 소문자화와 엄격한 형변환
- [DB별 SQL에서 조용히 틀리는 조건식 패턴](DB·SQL/DB별 SQL에서 조용히 틀리는 조건식 패턴.md) — Oracle || 와 - 우선순위, MySQL IN 안의 OR, BETWEEN 인자 순서, 날짜에 숫자 빼기, 시각 붙은 날짜 = 비교, ORDER BY 없는 순서 의존
- [MySQL 조인 컬럼 문자셋 불일치로 인덱스를 못 쓰는 문제](DB·SQL/MySQL 조인 컬럼 문자셋 불일치로 인덱스를 못 쓰는 문제.md) — utf8mb4·utf8 조인의 EXPLAIN 증상, 상수 조건이 상대 테이블로 안 넘어가는 경우와 EXISTS·조건 명시 우회, 앱 연결 콜레이션으로 재기, CONVERT 변형으로 미리 재기, 콜레이션까지 맞춘 ALTER 생성, 데이터 있을 때 잠금·인덱스 길이·사전 점검, 변환 후 통계와 ANALYZE
- [Oracle DB 링크와 원격 객체 존재 확인](DB·SQL/Oracle DB 링크와 원격 객체 존재 확인.md) — 링크 존재·접속·원격 객체·읽기 권한 단계별 확인, 단계별 ORA 오류 의미, 시노님, 링크 세션 닫기
- [페이징 목록 쿼리가 느린 원인과 개선 측정](DB·SQL/페이징 목록 쿼리가 느린 원인과 개선 측정.md) — 쓰지 않는 1:N 조인과 DISTINCT, 카운트 중복, 계산값 정렬, 인덱스를 고르지 않는 경우, STRAIGHT_JOIN 실험, 2단계 페이징 위험, 결과 해시로 전후 비교
- [JDBC batch와 다건 INSERT, 일부 실패 처리](DB·SQL/JDBC batch와 다건 INSERT, 일부 실패 처리.md) — 세 방식의 왕복·실패 동작, 드라이버별 batch 효과(rewriteBatchedStatements, 프로시저), 문장 단위 롤백과 커밋 한 번, 반으로 나눠 재시도, Oracle INSERT ALL 시퀀스 문제, 다건 프로시저 요청
- [PL-SQL 패키지 명세·본문과 프로시저 시그니처](DB·SQL/PL-SQL 패키지 명세·본문과 프로시저 시그니처.md) — 스펙·바디(선언부·구현부), 블록의 선언부·실행부·예외처리부, 시그니처·형식/실제 파라미터 용어, user_arguments 조회
- [MariaDB 뷰 DEFINER 계정과 콜레이션 오류(1449·1267)](DB·SQL/MariaDB 뷰 DEFINER 계정과 콜레이션 오류(1449·1267).md) — 계정 이름 대소문자, 잠긴 DEFINER 계정 추가, 뷰 상수가 만들 때 콜레이션으로 저장되는 문제와 재현, 예외 변환기 null로 NPE만 보이는 경우
- [인덱스를 타는 조건과 온라인 인덱스 추가(INPLACE·LOCK=NONE)](DB·SQL/인덱스를 타는 조건과 온라인 인덱스 추가(INPLACE·LOCK=NONE).md) — 함수로 감싼 컬럼을 범위 조건으로, 경계값 대량 대조, 용도별 복합 인덱스와 잠금 범위, 온라인 DDL 옵션의 안전장치 역할, 메타데이터 잠금, Galera TOI
- [MariaDB 버퍼 풀 크기를 재시작 없이 줄이기](DB·SQL/MariaDB 버퍼 풀 크기를 재시작 없이 줄이기.md) — SET GLOBAL로 바로 축소, 실제 캐시 사용량으로 크기 정하기, 동적·정적 변수 구분, 설정 파일 유지

## 네트워크
- [LDAP 연결 끊김 원인 구분](네트워크/LDAP 연결 끊김 원인 구분.md) — 오류 메시지별 의미, TTL 비교로 끊은 주체 찾기
- [SSH 공개키 인증과 비대화형 실행 옵션](네트워크/SSH 공개키 인증과 비대화형 실행 옵션.md) — ssh-copy-id와 Windows 대체 명령, BatchMode 등 -o 옵션, 종료 코드 255와 오류 구분
- [WAF 웹 애플리케이션 방화벽과 오탐 대응](네트워크/WAF 웹 애플리케이션 방화벽과 오탐 대응.md) — L7 검사와 룰셋, 정상 입력 403 오탐 증상, 서버 로그에 요청이 없으면 의심, 예외 등록·인코딩
- [HTTP Range 요청과 206 응답(미디어 재생 프록시)](네트워크/HTTP Range 요청과 206 응답(미디어 재생 프록시).md) — audio 태그의 구간 요청, iOS Safari 206 필수, 2단 프록시에서 상태코드·Content-Range 전달, RestTemplate 대신 직접 스트림 중계, 탭 즉시 재생
- [SSH config Host 별칭과 hosts 파일의 차이](네트워크/SSH config Host 별칭과 hosts 파일의 차이.md) — git 원격의 짧은 호스트명 정체, SSH만 해석되어 ping은 안 됨, ssh -G 확인, IP로 접속하면 별칭 블록의 ssh-rsa 허용이 빠져 no matching host key type

### 웹소켓
- [WebSocket 기본 개념과 Java·브라우저 문법](네트워크/웹소켓/WebSocket 기본 개념과 Java·브라우저 문법.md) — HTTP와 차이, 핸드셰이크·세션·프레임·ping/pong 용어, JSR-356 생명주기 어노테이션, 연결별 인스턴스와 Spring 빈, Configurator 쿠키·Origin, 동기·비동기 전송, 브라우저 API, STOMP·SockJS 관계
- [WebSocket 종료 코드로 끊긴 구간 찾기](네트워크/웹소켓/WebSocket 종료 코드로 끊긴 구간 찾기.md) — RFC 6455 종료 코드, 다단 중계에서 코드·사유 설계, 1회용 티켓, 프록시 Upgrade·유휴 타임아웃
- [L4·L7 로드밸런서와 WebSocket 설정 요청](네트워크/웹소켓/L4·L7 로드밸런서와 WebSocket 설정 요청.md) — L4와 Nginx(L7·stream) 차이, Upgrade·유휴 타임아웃·최대 연결 시간·아웃바운드, ping 주기, 트래픽 추정, SockJS 폴백
- [실시간 변경 알림 대신 제출 시 버전 검사](네트워크/웹소켓/실시간 변경 알림 대신 제출 시 버전 검사.md) — 웹소켓 알림이 맞지 않는 경우(정확성·인메모리 브로커·새로고침 손실), 낙관적 잠금, 수정 시각을 바꾸는 변경 확인, 시간대·형식 맞추기, opener 없는 팝업

## Windows·자동화
- [작업 스케줄러로 스크립트 자동 실행](Windows·자동화/작업 스케줄러로 스크립트 자동 실행.md) — UAC 없이 관리자 스크립트 실행, idle 감지, 창 없는 실행, 작업 잠시 끄기·다시 켜기
- [예약 재시작과 전원 설정](Windows·자동화/예약 재시작과 전원 설정.md) — shutdown /g와 ARSO, 절전 방지 powercfg
- [PowerShell과 Git Bash 사용 시 주의점](Windows·자동화/PowerShell과 Git Bash 사용 시 주의점.md) — curl 별칭, CRLF 변환, 스크립트 인코딩, 1건 배열, 실행 정책 Process 범위, 사용자 환경 변수 상속, 함수·별칭 충돌, properties \u 이스케이프, 폴더 이동 막힘
- [PowerShell 기초 문법(Java 개발자용)](Windows·자동화/PowerShell 기초 문법(Java 개발자용).md) — 변수 스코프, 단어 연산자, @ 기호, 객체 파이프, cmdlet 호출, 경로 명령
- [PowerShell GUI 도구에서 외부 프로세스 출력 실시간 표시](Windows·자동화/PowerShell GUI 도구에서 외부 프로세스 출력 실시간 표시.md) — 창 없는 실행과 출력 리다이렉트, DoEvents, 프로세스 트리 종료, exe로 감싸도 소스가 보이는 이유
- [HTML 파일을 작업 표시줄에 앱처럼 고정하기](Windows·자동화/HTML 파일을 작업 표시줄에 앱처럼 고정하기.md) — chrome --app 바로가기, PowerShell로 .lnk·아이콘 지정, 고정은 수동, 아이콘 캐시
- [netsh Wi-Fi 전환 배치 파일을 다른 PC에 나눠 줄 때](Windows·자동화/netsh Wi-Fi 전환 배치 파일을 다른 PC에 나눠 줄 때.md) — 고정 IP 충돌, %~dp0 상대 경로, PC 전용 바로가기·어댑터 이름, 평문 비밀번호 XML, 재시도 없는 연결, UTF-8 배치 한글 깨짐, DHCP 복귀
- [작업 표시줄 아이콘에 계단식 메뉴 붙이기](Windows·자동화/작업 표시줄 아이콘에 계단식 메뉴 붙이기.md) — Windows 11에서 없어진 도구 모음, 기본 csc.exe로 WinForms 빌드, 폴더 구조를 메뉴로(번호 순서·구분선), 고정과 .gitignore
- [시작프로그램을 꺼도 다시 켜지는 앱 정리](Windows·자동화/시작프로그램을 꺼도 다시 켜지는 앱 정리.md) — Run·StartupApproved·시작 폴더·스토어 앱 StartupTask, 앱 자체 "로그인 시 열기"가 등록을 되살리는 이유와 순서

## Claude Code
- [Remote Control로 휴대폰에서 이어 쓰기](Claude Code/Remote Control로 휴대폰에서 이어 쓰기.md) — 동작 구조, 켜는 방법, 끊기는 조건, 다른 기기에 세션 제목이 보이는 이유
- [하네스 엔지니어링 개념](Claude Code/하네스 엔지니어링 개념.md) — 모델을 뺀 나머지 전부, 에이전트 루프·도구·컨텍스트·메모리·가드레일·피드백 루프, 지침 파일과 스킬 나누기, 사용자 규칙과 AI 관리 문서 분리·변경 감지, 승인 후 동기화, 규칙 충돌 검사
- [지침과 지식 노트를 로딩 시점에 맞게 배치하기](Claude Code/지침과 지식 노트를 로딩 시점에 맞게 배치하기.md) — 기능별 로딩 시점, rules의 paths 지정 노트, 남길 것과 뺄 것, 로드 확인 방법
- [권한 규칙으로 특정 명령만 자동 허용하기](Claude Code/권한 규칙으로 특정 명령만 자동 허용하기.md) — ask가 allow보다 우선, 입력 검사 래퍼만 allow, 자동 허용을 거둘 때의 함정, 지침 파일 보호의 빈틈
- [MCP 서버 도입 판단과 Serena(jdtls)](Claude Code/MCP 서버 도입 판단과 Serena(jdtls).md) — MCP 장점, 망 분리 환경에서 안 맞는 이유, 로컬 DB 읽기 전용 조건, jdtls·Serena와 grep 비교
- [훅(hook) 작성 요령](Claude Code/훅(hook) 작성 요령.md) — 빠르게 끝내는 방법, 훅 출력이 컨텍스트로 들어가는 점, SubagentStart/Stop과 Stop 훅에 끊기는 백그라운드 서브에이전트, PowerShell 5.1 입력 인코딩(CP949)으로 JSON 해석 실패, 도구 호출마다 도는 훅의 비용 줄이기(동기 훅 하나·백그라운드 기록·작은 상태 파일)
- [예약 작업 무인 실행과 권한 규칙](Claude Code/예약 작업 무인 실행과 권한 규칙.md) — 고정 스크립트와 좁은 허용 규칙, 놓친 날 보충 장치의 구멍(UserPromptSubmit 훅으로 무인 세션 구분), 권한 요청 대기로 멈춘 실행, 목차 기반 요약
- [로컬 환경 전환 불가 메시지](Claude Code/로컬 환경 전환 불가 메시지.md) — 메시지가 나오는 경우, + 버튼 세션은 실제로 로컬 실행, 재시작·폴더 재지정·정책 확인 순서
- [Agent Skills SKILL.md 위치(Claude Code와 Codex)](Claude Code/Agent Skills SKILL.md 위치(Claude Code와 Codex).md) — SKILL.md 표준, .claude/skills와 .agents/skills 차이, AGENTS.md에 핵심 규칙 두기, 규칙·절차 분리 기준, CLAUDE.md import, 폴더 링크
- [커밋 메시지 Co-Authored-By 줄 끄기(attribution 설정)](Claude Code/커밋 메시지 Co-Authored-By 줄 끄기(attribution 설정).md) — settings.json attribution 항목, 적용 범위, push 전 커밋 메시지에서 줄 빼기와 diff 확인
- [Claude Code 세션에서 Codex CLI로 교차 리뷰하기](Claude Code/Claude Code 세션에서 Codex CLI로 교차 리뷰하기.md) — 바깥 스크립트 방식의 문제(훅·비밀 diff·read-only 한계), 세션 안 스킬 절차와 판정표, 고른 diff만 보내기, 과정 보이게 하기, Windows 실행 정책·샌드박스 전역 설치 함정
- [한 계정으로 여러 PC에서 Claude Code 쓰기](Claude Code/한 계정으로 여러 PC에서 Claude Code 쓰기.md) — PC별 인증, 공유 사용량 한도, PC마다 따로인 기록·설정, 약관
- [Claude 데스크톱 앱 메모리 사용량과 세션 보관](Claude Code/Claude 데스크톱 앱 메모리 사용량과 세션 보관.md) — 세션 프로세스·렌더러·터미널 탭별 크기, 보관이 하는 일(탭 속 서버도 종료), 오래된 세션은 메모리 무관, 줄이는 순서

## 기타
- [JavaScript var 스코프와 중첩 루프 버그](기타/JavaScript var 스코프와 중첩 루프 버그.md) — 함수 스코프로 루프 변수가 공유되는 원인과 증상, 수정 방법
- [Docker 컨테이너 파일을 호스트로 꺼내기](기타/Docker 컨테이너 파일을 호스트로 꺼내기.md) — docker cp 후 SFTP로 받기, 볼륨 마운트 경로 확인
- [앱·사이트 사용 제한 프로그램의 구현 방식과 한계](기타/앱·사이트 사용 제한 프로그램의 구현 방식과 한계.md) — Windows 프로세스 감시, 설정 파일 잠금·레지스트리 사본으로 초기화 우회 막기, Android 접근성 서비스, 기본 제공 기능과 비교
- [iframe src 변경과 브라우저 방문 기록](기타/iframe src 변경과 브라우저 방문 기록.md) — 뒤로가기를 여러 번 눌러야 하는 원인, location.replace, history.length로 추적
- [외부 검색엔진 연동과 검색 결과 권한 필터링](기타/외부 검색엔진 연동과 검색 결과 권한 필터링.md) — 색인용 뷰, 트리거 증분 색인, 권한 색인과 접근 식별자 목록 대조
- [톰캣 폴더 교체 배포 스크립트 설계](기타/톰캣 폴더 교체 배포 스크립트 설계.md) — 준비·교체 분리, 준비 완료 표시 파일, mkdir 잠금, ps -o 함정, 기동 확인과 되돌리기, JSP 컴파일 캐시(work) 삭제, pgrep -f 범위 좁히기, 비어 있는 외부 환경변수 경로, 늦게 내려가는 톰캣
- [목록 화면에서 선택 행과 전역 변수가 어긋나는 버그](기타/목록 화면에서 선택 행과 전역 변수가 어긋나는 버그.md) — 체크박스·Ctrl 클릭이 변수를 갱신하지 않아 다른 항목이 처리되는 원인, 행 안 드롭다운이 선택 행을 고치는 경우, 마우스로는 안 나고 키보드·selectOption에서만 나는 이유, 수정과 재현 요령
- [외부 응답을 화면에 넣는 방식(텍스트 노드와 innerHTML)](기타/외부 응답을 화면에 넣는 방식(텍스트 노드와 innerHTML).md) — 태그가 글자로 보이는 이유, 컨트롤 노출, data-* 속성과 경로별 필터
- [nohup으로 jar 백그라운드 실행과 java 옵션 순서](기타/nohup으로 jar 백그라운드 실행과 java 옵션 순서.md) — -D·-cp는 -jar 앞에, -jar이면 -cp 무시, nohup 출력 처리와 기동 로그
- [HWP·PDF 문서에서 텍스트 뽑기](기타/HWP·PDF 문서에서 텍스트 뽑기.md) — hwplib·pyhwp, pdftotext의 한글 CID 글꼴(poppler-data) 누락, 추출 결과의 한계
- [날짜 기준 전송 배치의 재전송 설계](기타/날짜 기준 전송 배치의 재전송 설계.md) — 이력 날짜의 의미 맞추기, 행 없으면 생성, 서버 검증, 전송 플래그 되돌리기, 날짜 계산 한 곳에서, 갱신 키 유지
- [파일 경로만 넘기는 서버 간 연동과 공유 저장소](기타/파일 경로만 넘기는 서버 간 연동과 공유 저장소.md) — 공유가 깨지면 200 + 빈 결과, 마운트·기준 경로·권한 확인, 로컬 개발 서버의 구조적 한계
- [ps STAT 값(Ss+)과 남아 있는 tail -F 프로세스](기타/ps STAT 값(Ss+)과 남아 있는 tail -F 프로세스.md) — STAT 문자 의미, SSH로 띄운 tail이 세션 리더가 되는 이유, grep에 걸리는 로그 경로
- [docker exec 창을 닫아도 컨테이너 안 프로세스가 남는 이유](기타/docker exec 창을 닫아도 컨테이너 안 프로세스가 남는 이유.md) — -t 가상 터미널과 containerd-shim, 신호가 전달되지 않는 구조, -i와 EOF 감시로 tail 정리, ServerAliveInterval
- [브라우저 마이크 녹음 기능을 붙일 때 확인할 것](기타/브라우저 마이크 녹음 기능을 붙일 때 확인할 것.md) — https 보안 컨텍스트, 앱 웹뷰 권한, 화면 이탈 시 마이크 해제, 다단 WebSocket 중계와 단기 티켓, 프록시 Upgrade·고정 세션
- [HTML select 펼친 목록에 CSS가 안 먹는 이유](기타/HTML select 펼친 목록에 CSS가 안 먹는 이유.md) — 네이티브 옵션 목록, jQuery Mobile 자체 목록, 둥근 팝업에서 글자 잘림
- [시간대 오프셋 형식 불일치로 생기는 기간 판정 오류](기타/시간대 오프셋 형식 불일치로 생기는 기간 판정 오류.md) — 형식이 다르면 조용히 UTC 반환, 시차만큼 기간 밖 판정, 정규화와 대조 테스트, 같은 함수의 다른 호출 경로 점검, 404·403으로 나타나는 경우
- [Chrome unload 이벤트 폐지와 Permissions policy 경고](기타/Chrome unload 이벤트 폐지와 Permissions policy 경고.md) — 경고의 의미, 라이브러리 출처 찾기, pagehide·visibilitychange·sendBeacon 대체
- [audio 태그로 mp4 재생과 MIME 타입](기타/audio 태그로 mp4 재생과 MIME 타입.md) — audio/mp4 값, 206 응답, 코덱별 재생 차이, 허용 목록 한 곳 관리, 전사 서버 지원·용량
- [외부 API가 빈 응답을 줄 때 원인 가르기](기타/외부 API가 빈 응답을 줄 때 원인 가르기.md) — 공통 실패 안내의 원인 분리, 테스트 요청의 실제 재현 여부, 응답 시간 단서, 입력 조합 표, 화면 값과 저장 값 차이
- [끼워 넣은 span이 주변 글꼴을 따르지 않는 이유](기타/끼워 넣은 span이 주변 글꼴을 따르지 않는 이유.md) — 태그 전역 CSS가 상속을 막는 원리, 표시 클래스에만 inherit
- [화면 안내 메시지 설계(토스트와 알림창, 중복 알림)](기타/화면 안내 메시지 설계(토스트와 알림창, 중복 알림).md) — 성공은 토스트·실패는 알림창, 하위 함수는 결과만 반환, 결과 확인 후 상태 변경, 일괄 처리 안내
- [window.open 창 이름 재사용과 opener](기타/window.open 창 이름 재사용과 opener.md) — 같은 이름 창 재로드로 입력 내용 소실, 고정 이름 보조 창의 opener, 새 탭·위젯 창의 opener 없음과 postMessage 열기 요청, 비동기 뒤 팝업 차단과 빈 창 먼저 열기
- [목록과 건수를 따로 요청할 때 응답 순서 버그](기타/목록과 건수를 따로 요청할 때 응답 순서 버그.md) — 순서가 뒤집히는 이유, 페이지 번호 소실·이전 건수 사용·재사용 파라미터에 남은 플래그, readyState 확인, route 지연으로 재현
- [jQuery Mobile 화면에서 무거운 라이브러리 지연 로드](기타/jQuery Mobile 화면에서 무거운 라이브러리 지연 로드.md) — AJAX 이동이 head를 버리는 점, 활성 페이지 범위 선택자, 안 쓰는 의존 파일 찾기, 1회 로드·진행 중 공유·script 태그 방식(CSP)·타임아웃, 나간 뒤 그리지 않기
- [개발자 도구로 iframe 화면 이동 시간 재기](기타/개발자 도구로 iframe 화면 이동 시간 재기.md) — Finish가 안 나오는 이유, End Time 정렬·Performance 스크린샷과 설정·읽는 법, 프레임 넘는 콘솔 스니펫, 일반 사용자용 측정 북마클릿(요청 추적·DOM 정지·주기 조회 제외), 측정 도구 타당성 교차 검증, Playwright, access log %D
- [Playwright 화면 자동화에서 자주 막히는 지점](기타/Playwright 화면 자동화에서 자주 막히는 지점.md) — 숫자로 시작하는 id 선택자, window.event 핸들러와 실제 클릭, 다시 그려지는 목록, 확인 창 골라 답하기, 대기 시간, 건별 예외와 정리, 네트워크 재연결, 셸로 넣은 정규식 깨짐
- [Redmine REST API로 일감 추출·검색·일괄 변경](기타/Redmine REST API로 일감 추출·검색·일괄 변경.md) — 상세·첨부 받기, 화면 필터 그대로 쓰는 검색과 100건 한도, 워크플로 무시·422와 재조회 확인, seen 값을 넣은 오프라인 변경 요청 파일, 다른 Redmine 복제의 한계와 단방향 동기화 규칙, 이름 맞추기·신규→모든 상태 워크플로, 권한 없으면 일반 댓글로 올라가는 비공개 댓글, 프로젝트 설정 API 가능 범위와 반자동 생성
- [정적 HTML에서 로컬 폴더 읽기(File System Access API)](기타/정적 HTML에서 로컬 폴더 읽기(File System Access API).md) — file:// 페이지의 한계, showDirectoryPicker 재귀 읽기, 핸들 IndexedDB 저장과 다시 허용, webkitdirectory 대체, 최신본만 남기기·새 항목 표시
- [달력 화면의 중복 동기 요청 없애기와 받은 응답 재사용](기타/달력 화면의 중복 동기 요청 없애기와 받은 응답 재사용.md) — 인자 대신 DOM에서 기간을 읽어 생기는 같은 요청 2회, 첫 응답으로 바로 그리기, 같은 달 클릭은 요청 0회, 지연 콜백이 다른 화면을 읽는 버그
- [가로 메뉴 넘침 처리(폭을 재서 검색창 줄이고 메뉴 숨기기)](기타/가로 메뉴 넘침 처리(폭을 재서 검색창 줄이고 메뉴 숨기기).md) — absolute 입력창 겹침 원인, 넘친 메뉴가 표시 없이 사라지는 문제, 드래그 패널에서 미디어쿼리 대신 실측, 축소→숨김→… 단계, 여백 포함 측정, ResizeObserver·rAF
- [MIDI 악보와 음원 재생을 맞춰 현재 음표 표시하기](기타/MIDI 악보와 음원 재생을 맞춰 현재 음표 표시하기.md) — mp3 자동 채보가 어려운 이유, tick→초 시각표, 카운트인과 피치 비교로 시작 오프셋 맞추기, 템포 흔들리는 녹음 한계, 재생 마디 기준 페이지 넘김, 반주+녹음 지연 보정
- [정적 웹앱을 GitHub Pages PWA로 배포하고 홈 화면에 추가하기](기타/정적 웹앱을 GitHub Pages PWA로 배포하고 홈 화면에 추가하기.md) — Pages 무료 한도·약관, manifest·서비스 워커 오프라인, iOS 홈 화면 앱 저장소 분리, 웹뷰 APK와 한글 경로 Gradle 실패·JDK 17, 스토어 비용
