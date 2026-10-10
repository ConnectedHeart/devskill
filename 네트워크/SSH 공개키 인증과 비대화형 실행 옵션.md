# SSH 공개키 인증과 비대화형 실행 옵션

> 분류: 네트워크 · 최근 갱신: 2026-10-08

## 한 줄 요약
공개키를 서버에 한 번 등록하면 비밀번호 없이 접속할 수 있고, 스크립트에서 ssh를 쓸 때는 `-o` 옵션으로 "사람에게 묻는 상황"을 모두 없애야 멈추지 않는다.

## 내용
- **원리**: PC의 공개키(`.pub`)를 서버 계정의 `~/.ssh/authorized_keys`에 넣어 두면, 접속할 때 짝이 되는 개인키로 인증한다. 등록은 서버와 계정마다 한 번씩 필요하다.
- **등록 도구**: `ssh-copy-id`가 이 일을 해 준다. Linux·Mac·Git Bash에는 있지만 Windows의 cmd·PowerShell에는 없어서 같은 일을 하는 한 줄 명령으로 대신한다.
- **기본 키 이름**: `id_ed25519`, `id_rsa` 같은 기본 이름이면 ssh가 알아서 찾는다. 다른 이름의 키는 `-i 개인키경로`로 지정한다. 지정하는 것은 `.pub`이 없는 개인키다.
- **`-o 설정=값`**: `~/.ssh/config`에 적을 설정을 그 실행 한 번에만 준다. 자주 쓰는 것만 `-p`(포트)처럼 짧은 옵션이 있고 나머지는 모두 `-o`다.
- **자동화에 쓰는 세 가지**

| 설정 | 의미 | 없으면 |
|---|---|---|
| `BatchMode=yes` | 비밀번호 등 어떤 입력도 묻지 않고 바로 실패 | 키 인증이 안 될 때 입력을 기다리며 멈춤 |
| `StrictHostKeyChecking=accept-new` | 처음 보는 서버는 묻지 않고 등록, 키가 바뀐 서버는 거부 | 첫 접속 때 yes/no 질문에서 멈춤 |
| `ConnectTimeout=10` | 10초 안에 연결되지 않으면 포기 | 서버가 꺼져 있으면 오래 기다림 |

- **종료 코드**: ssh 자체가 접속에 실패하면 255로 끝난다. 그 외의 값은 원격 명령의 종료 코드다. 스크립트에서 "접속 실패"와 "명령 실패"를 이것으로 구분할 수 있다.
- **오류 메시지 구분**
  - `No route to host`: 서버까지 가는 네트워크 경로가 없음(다른 망, 서버 꺼짐). 키나 비밀번호 이전의 문제다.
  - `Connection refused`: 서버에는 닿았지만 그 포트에서 받아 주는 프로그램이 없음(포트 오류 등).
  - `Permission denied (publickey)`: 닿았지만 키 인증이 안 됨.

## 예시
```bash
# 키 만들기 (없을 때)
ssh-keygen -t ed25519

# 등록 - Linux, Mac, Git Bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub -p 22 user@example.com

# 등록 - Windows cmd
type %USERPROFILE%\.ssh\id_ed25519.pub | ssh user@example.com "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"

# 확인: 비밀번호를 묻지 않고 호스트 이름이 나오면 성공
ssh -o BatchMode=yes user@example.com hostname

# 스크립트에서 쓰는 형태
ssh -p 22 -o StrictHostKeyChecking=accept-new -o ConnectTimeout=10 -o BatchMode=yes user@example.com "명령"
```

## 주의할 점
- 키 인증은 SSH 로그인만 해결한다. 원격에서 `sudo`가 비밀번호를 묻는다면 그건 따로 풀어야 한다.
- 키에 암호(passphrase)가 걸려 있으면 접속할 때마다 묻는다. 자동화에는 암호 없는 전용 키를 쓰거나 `ssh-agent`에 올려 둔다.
- 서버 설정에서 `PubkeyAuthentication`이 꺼져 있으면 등록해도 비밀번호를 묻는다.
- 개인키가 유출되면 그 키가 등록된 모든 서버에 접속할 수 있다. 개인키 파일은 공유하거나 복사해 두지 않는다.
- 원격 명령을 작은따옴표로 감싸 보내면 원격 셸이 `~`를 홈으로 바꾸지 않는다. 경로는 전체 경로로 적는 편이 안전하다.

## 참고
- OpenSSH 매뉴얼: ssh(1), ssh_config(5), ssh-copy-id(1)
