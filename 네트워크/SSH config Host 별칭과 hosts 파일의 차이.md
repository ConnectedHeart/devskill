# SSH config Host 별칭과 hosts 파일의 차이

> 분류: 네트워크 · 최근 갱신: 2026-11-06

## 한 줄 요약
git 원격 주소에 보이는 짧은 호스트명이 어디서도 안 풀리는 것 같다면 `~/.ssh/config`의 `Host` 별칭일 수 있다. SSH만 해석하므로 `ping`이나 브라우저에서는 그 이름을 쓸 수 없다.

## 내용
### 두 가지 이름 해석
| 구분 | hosts 파일 (`/etc/hosts`, `C:\Windows\System32\drivers\etc\hosts`) | `~/.ssh/config` |
|---|---|---|
| 적용 범위 | OS 전체 (모든 프로그램) | ssh, scp, sftp, git(ssh 원격) 등 SSH 클라이언트만 |
| 바꿀 수 있는 것 | 이름 → IP | 이름 → IP/호스트명, 포트, 사용자, 키 파일, 기타 SSH 옵션 |
| 권한 | 관리자 | 사용자 자신 |

### SSH config 예
```
Host gitbox
    HostName 192.0.2.10
    User git
    Port 22
    IdentityFile ~/.ssh/id_rsa_work
```
- `git remote add origin git@gitbox:/srv/git/repo.git`처럼 쓰면 SSH가 `gitbox`를 위 설정으로 풀어 192.0.2.10:22에 `git` 사용자와 지정한 키로 접속한다.
- hosts에 `gitbox`가 없어도 `git fetch`/`push`는 된다. 반대로 `ping gitbox`는 "호스트를 찾을 수 없음"이 난다.
- 같은 서버를 IP로 직접 쓴 원격(`user@192.0.2.10:...`)과 별칭으로 쓴 원격이 섞여 있으면 커밋 메시지의 병합 출처가 다르게 보일 뿐 같은 서버다.

### 확인 방법
```bash
cat ~/.ssh/config
ssh -G gitbox | grep -E '^(hostname|user|port|identityfile)'   # 최종 적용 값
git remote -v
```

## 핵심 코드
### 1) 별칭 정의 (~/.ssh/config)
```ssh-config
# ssh_config는 줄 끝 주석을 지원하지 않으므로 주석은 따로 한 줄로 쓴다
Host gitbox
    # 실제 주소로 바꾼다
    HostName 192.0.2.10
    User git
    Port 22
    IdentityFile ~/.ssh/id_rsa_work
    # 지정한 키만 시도
    IdentitiesOnly yes
```

### 2) 원격을 별칭으로 바꾸고 접속 확인
```bash
git remote set-url origin git@gitbox:/srv/git/repo.git
git ls-remote origin                 # 목록이 나오면 별칭·키·포트가 모두 맞음
ssh -G gitbox | grep -E '^(hostname|user|port|identityfile) '
```
```powershell
ssh -G gitbox | Select-String '^(hostname|user|port|identityfile) '
```

### 3) config 없이 한 번만 키 지정
```bash
GIT_SSH_COMMAND="ssh -i ~/.ssh/id_rsa_work -o IdentitiesOnly=yes" git fetch origin
```

### 4) 비교: OS 전체에 이름을 주려면 hosts 파일 (관리자 권한)
```text
# /etc/hosts 또는 C:\Windows\System32\drivers\etc\hosts
192.0.2.10    gitbox
```
```powershell
[System.Net.Dns]::GetHostAddresses('gitbox')   # hosts·DNS로 풀리는지 (ssh config는 보지 않음)
```

### 5) 별칭 블록 설정은 IP로 접속할 때 적용되지 않는다
옛 서버가 `ssh-rsa`·`ssh-dss` 호스트 키만 내놓으면 최신 OpenSSH(8.8 이상)는 기본으로 거부한다. 그래서 별칭 블록에 허용 옵션을 넣어 두는데, 같은 서버를 **IP로** 부르면 그 블록이 매칭되지 않아 아래 오류가 난다.
```text
Unable to negotiate with 192.0.2.10 port 22: no matching host key type found. Their offer: ssh-rsa,ssh-dss
```
`Host` 줄에 IP도 같이 적으면 이름·IP 어느 쪽으로 접속해도 같은 설정이 적용된다.
```ssh-config
# 별칭과 IP를 함께 매칭
Host gitbox 192.0.2.10
    HostName 192.0.2.10
    User git
    HostKeyAlgorithms +ssh-rsa
    PubkeyAcceptedAlgorithms +ssh-rsa
```
```bash
ssh -G 192.0.2.10 | grep -iE '^(hostkeyalgorithms|user) '   # IP 접속에도 ssh-rsa가 들어가는지
```

## 주의할 점
- `ssh-dss`는 OpenSSH 10에서 완전히 빠졌다. 서버가 `ssh-rsa`도 내놓으면 `ssh-rsa`만 허용해도 접속된다.
- IP 블록에 `User`를 넣으면 IP로 접속할 때 계정을 생략해도 그 계정이 쓰인다. 명령에 `다른계정@IP`로 적으면 그쪽이 우선한다.
- IDE의 clone 화면 등에서 원격 주소를 IP로 붙여 넣으면 별칭 설정을 타지 않는다. 오류가 특정 경로에서만 나면 원격 주소 형식부터 본다.
- `IdentityFile`에 적은 키가 없거나 권한이 넓으면 접속이 실패한다. 키 파일을 옮겼다면 config도 같이 고친다.
- Windows에서 config 파일은 확장자 없이 `config`이고 줄 끝은 LF여도 CRLF여도 동작한다. 경로는 `C:/Users/...` 형식도 된다.

## 참고
- OpenSSH: ssh_config(5), `ssh -G`
