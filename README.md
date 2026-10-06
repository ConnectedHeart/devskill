# 클로드 지식 내용

Claude와 작업하면서 알게 된 기술 지식을 분류별·주제별로 정리한 노트입니다. 특정 회사나 프로젝트에 한정된 내용은 빼고, 다른 곳에서도 쓸 수 있는 내용만 담습니다.

- 구조: `<분류>/<주제명>.md`
- 같은 주제는 한 파일에 계속 보탭니다.

## Git
- [git bundle 생성과 병합](Git/git bundle 생성과 병합.md) — 저장소를 파일 하나로 옮기는 방법, FETCH_HEAD로 병합하는 이유, --no-ff
- [cherry-pick 충돌 해결과 검증](Git/cherry-pick 충돌 해결과 검증.md) — 선행 커밋과 순서, worktree 시험, 원 커밋과 대조하는 검증 절차
- [병합 커밋에서 사라진 코드 찾기](Git/병합 커밋에서 사라진 코드 찾기.md) — 같은 부모를 재병합해 비교, 지워짐 판정 기준, 복구와 이력 맞추기

## 인증·보안
- [Keycloak audience(aud)와 azp 검증](인증·보안/Keycloak audience(aud)와 azp 검증.md) — aud·azp의 의미, Audience 매퍼 추가, API 서버 검증과 적용 순서
- [1회용 코드 SSO와 refresh token 로테이션](인증·보안/1회용 코드 SSO와 refresh token 로테이션.md) — 토큰을 브라우저로 내보내지 않는 SSO 흐름, 오류 위치 구분, 로테이션 충돌
- [Keycloak 토큰 클레임(jti·sid·sub)과 로그아웃 범위](인증·보안/Keycloak 토큰 클레임(jti·sid·sub)과 로그아웃 범위.md) — 재발급 시 바뀌는 값, 세션 단위 로그아웃, 자체 검증 서버의 틈
- [토큰 보관 위치와 재발급 기준](인증·보안/토큰 보관 위치와 재발급 기준.md) — 메모리·쿠키·세션·DB 비교, 수명 절반 기준, 토큰 응답 캐시 금지, 쿠키 공유 범위

## Java·Spring
- [외부 API 판정 결과의 로컬 캐시 설계](Java·Spring/외부 API 판정 결과의 로컬 캐시 설계.md) — 성공·실패 캐시, request 속성과의 역할 구분, 다중 서버 한계
- [Future로 외부 호출 전체 시간 제한하기](Java·Spring/Future로 외부 호출 전체 시간 제한하기.md) — connect/read 타임아웃의 빈틈, ExecutionException 벗기기, cancel의 한계
- [RestTemplate JSON 호출의 숫자 타입과 타임아웃](Java·Spring/RestTemplate JSON 호출의 숫자 타입과 타임아웃.md) — Gson·json-simple·Jackson의 타입 차이, 전용 RestTemplate
- [코드 점검에서 자주 나오는 결함 패턴](Java·Spring/코드 점검에서 자주 나오는 결함 패턴.md) — 순회 중 삭제, 예외 삼킴, 확장자 판별, 프록시를 거치지 않는 내부 호출

## DB·SQL
- [AUTO_INCREMENT가 건너뛰며 증가하는 이유](DB·SQL/AUTO_INCREMENT가 건너뛰며 증가하는 이유.md) — auto_increment_increment와 Galera, LAST_INSERT_ID와 MAX의 차이

## 네트워크
- [LDAP 연결 끊김 원인 구분](네트워크/LDAP 연결 끊김 원인 구분.md) — 오류 메시지별 의미, TTL 비교로 끊은 주체 찾기

## Windows·자동화
- [작업 스케줄러로 스크립트 자동 실행](Windows·자동화/작업 스케줄러로 스크립트 자동 실행.md) — UAC 없이 관리자 스크립트 실행, idle 감지, 창 없는 실행
- [예약 재시작과 전원 설정](Windows·자동화/예약 재시작과 전원 설정.md) — shutdown /g와 ARSO, 절전 방지 powercfg
- [PowerShell과 Git Bash 사용 시 주의점](Windows·자동화/PowerShell과 Git Bash 사용 시 주의점.md) — curl 별칭, CRLF 변환, 스크립트 인코딩, 1건 배열

## Claude Code
- [Remote Control로 휴대폰에서 이어 쓰기](Claude Code/Remote Control로 휴대폰에서 이어 쓰기.md) — 동작 구조, 켜는 방법, 끊기는 조건
- [훅(hook) 작성 요령](Claude Code/훅(hook) 작성 요령.md) — 빠르게 끝내는 방법, 훅 출력이 컨텍스트로 들어가는 점
- [예약 작업 무인 실행과 권한 규칙](Claude Code/예약 작업 무인 실행과 권한 규칙.md) — 고정 스크립트와 좁은 허용 규칙, 놓친 날 보충 장치의 구멍, 목차 기반 요약

## 기타
- [JavaScript var 스코프와 중첩 루프 버그](기타/JavaScript var 스코프와 중첩 루프 버그.md) — 함수 스코프로 루프 변수가 공유되는 원인과 증상, 수정 방법
