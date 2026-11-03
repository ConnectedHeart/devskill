# netsh Wi-Fi 전환 배치 파일을 다른 PC에 나눠 줄 때

> 분류: Windows·자동화 · 최근 갱신: 2026-11-02

## 한 줄 요약
`netsh wlan`·`netsh interface ip`로 Wi-Fi와 고정 IP를 바꾸는 배치 파일은 만든 PC의 경로·IP·어댑터 이름·예약 작업에 묶여 있기 쉬우므로, 나눠 주기 전에 상대 경로로 바꾸고 사람마다 바꿀 값을 안내해야 한다.

## 내용
### 그대로 주면 깨지는 것
- **고정 IP:** 같은 IP를 두 PC가 쓰면 둘 다 연결이 끊긴다. 받는 사람마다 자기 IP로 바꿔야 한다(마스크·게이트웨이·DNS는 공통일 수 있음).
- **프로필 XML 절대 경로:** `netsh wlan add profile filename="C:\Users\<나>\..."`는 다른 PC에서 실패한다. 배치 파일이 있는 폴더를 뜻하는 `%~dp0`를 쓴다.
- **이 PC에만 있는 것:** 예약 작업을 부르는 바로가기, `C:\ProgramData` 등 폴더 밖 스크립트, 사용자 경로를 가리키는 `.lnk`는 받은 PC에서 아무 일도 하지 않는다.
- **어댑터 이름:** `"Wi-Fi"`로 고정하면 USB 무선랜(`Wi-Fi 2`) PC에서 IP 설정이 실패한다. `netsh interface show interface`로 이름을 확인하게 안내한다.

### 알아 둘 것
- 프로필을 `netsh wlan export profile key=clear`로 뽑은 XML에는 비밀번호가 평문(`protected=false`)으로 들어 있다. 내부에만 배포한다.
- 숨김 SSID는 첫 연결이 실패할 때가 있는데, 배치 파일이 재시도 없이 몇 초 뒤 IP를 바꾸면 "연결 안 된 채 고정 IP"가 된다. 재시도가 필요하면 PowerShell로 SSID를 확인하며 반복한다.
- UTF-8로 저장한 배치 파일은 cmd가 시스템 코드 페이지(한국어 Windows는 CP949)로 읽어 한글 출력이 깨진다. `chcp 65001`을 넣거나 ANSI로 저장한다.
- 고정 IP가 남아 있으면 집·외부 와이파이에서 인터넷이 안 된다. 그만 쓸 때 DHCP로 되돌리는 배치를 같이 준다.
- 관리자 권한이 필요하다(UAC 창에서 [예]).
- 사용 안내는 받는 PC에 Markdown 뷰어가 없을 수 있으니 txt로 둔다.

## 핵심 코드
### 내부망 전환 (고정 IP)
```bat
@echo off
chcp 65001 >nul
rem 프로필 XML은 이 배치 파일과 같은 폴더에 둔다
netsh wlan add profile filename="%~dp0inner_wifi.xml" user=all
netsh wlan connect name="<내부 SSID>" interface="Wi-Fi"
timeout /t 3 >nul
rem 아래 IP를 각자 배정받은 주소로 바꾼다
netsh interface ip set address name="Wi-Fi" static 192.0.2.50 255.255.0.0 192.0.2.1
netsh interface ip set dns name="Wi-Fi" static 192.0.2.10
```

### 외부망 전환 (DHCP 복귀)
```bat
@echo off
netsh wlan add profile filename="%~dp0outer_wifi.xml" user=all
netsh wlan connect name="<외부 SSID>" interface="Wi-Fi"
netsh interface ip set address name="Wi-Fi" dhcp
netsh interface ip set dns name="Wi-Fi" dhcp
```

### 확인 명령
```bat
netsh interface show interface          & rem 어댑터 이름
netsh wlan show interfaces              & rem 연결된 SSID
netsh wlan export profile name="<SSID>" folder="%~dp0"   & rem key=clear를 붙이면 비밀번호가 평문으로 들어감
```

## 주의할 점
- 배포본은 원본 폴더를 복사해 따로 만들고, 원본은 자동화(예약 작업)가 읽는 파일을 그대로 둔다.
- 다른 PC에서 실제로 실행해 보기 전에는 "동작 확인"으로 안내하지 않는다.

## 참고
- Microsoft 문서: Netsh commands for WLAN, netsh interface ip
- [작업 스케줄러로 스크립트 자동 실행](작업 스케줄러로 스크립트 자동 실행.md)
