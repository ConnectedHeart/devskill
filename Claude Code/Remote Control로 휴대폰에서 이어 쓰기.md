# Remote Control로 휴대폰에서 이어 쓰기

> 분류: Claude Code · 최근 갱신: 2026-09-30

## 한 줄 요약
Remote Control은 PC에서 돌고 있는 Claude Code 세션을 휴대폰 앱이나 웹에서 이어 쓰는 기능으로, 실행과 파일 접근은 PC에서 하고 대화만 동기화된다.

## 내용
### 동작 구조
- 휴대폰과 PC가 직접 연결되는 것이 아니라 둘 다 Anthropic 서버에 붙어 메시지를 주고받는다.
- 그래서 PC가 인터넷에 연결되어 있고 깨어 있어야 한다. PC가 인터넷이 안 되는 망에 있거나 절전 상태면 휴대폰에서 보낸 메시지가 도착하지 않는다.
- 명령 실행, 파일 읽기·쓰기는 모두 PC에서 일어난다.

### 사용 조건
- Pro / Max / Team / Enterprise 플랜, claude.ai 계정으로 로그인
- Team / Enterprise는 관리자가 허용해야 한다
- Zero Data Retention 조직은 사용할 수 없다

### 켜는 방법
- 세션 안에서 `/remote-control` (줄여서 `/rc`)
- 시작할 때 `claude --remote-control "<세션 이름>"`
- 서버 모드 `claude remote-control` (스페이스바로 QR 코드 표시)
- 데스크톱 앱: 설정 → Claude Code → "Connect new sessions to Remote Control"
- 항상 켜기: 설정에 `remoteControlAtStartup: true`

### 휴대폰에서
- Claude 앱 → Code 탭 → 초록 점이 있는 세션을 연다.
- 권한 확인 창도 휴대폰에 뜬다. 일정 시간 응답하지 않으면 거절로 처리된다.
- 일부 명령(`/plugin`, `/resume` 등)은 PC에서만 된다.
- 휴대폰에서 **새 세션을 시작**하려면 "휴대폰과 claude.ai에서 이 컴퓨터를 사용" 설정이 추가로 필요하다. 이미 열려 있는 세션을 이어 쓰는 데는 필요 없다.

## 주의할 점
- PC가 절전에 들어가면 끊긴다. 화면 잠금과 화면 꺼짐은 괜찮다. 전원 설정은 [예약 재시작과 전원 설정](../Windows·자동화/예약 재시작과 전원 설정.md) 참고.
- 휴대폰에서는 PC의 네트워크 상태를 바꿀 수 없다. PC가 인터넷이 끊기는 망으로 넘어가면 그 뒤 메시지는 전달되지 않으므로, 망을 잠깐 바꿔야 하는 작업은 "전환 → 실행 → 복귀"를 한 명령으로 묶어 PC에서 끝까지 실행되게 한다.
- PC를 재시작하면 이전 세션은 살아나지 않는다. 앱이 켜져 있어야 휴대폰에서 새 세션을 시작할 수 있다.
- 망분리 환경에서는 원격 전원 제어 장치(WoL, 스마트 플러그 등)가 보안 정책과 충돌할 수 있다.

## 참고
- Claude Code 문서: Remote Control
