# SSH config Host 별칭과 hosts 파일의 차이

> 분류: 네트워크 · 최근 갱신: 2026-10-27

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

## 주의할 점
- `IdentityFile`에 적은 키가 없거나 권한이 넓으면 접속이 실패한다. 키 파일을 옮겼다면 config도 같이 고친다.
- Windows에서 config 파일은 확장자 없이 `config`이고 줄 끝은 LF여도 CRLF여도 동작한다. 경로는 `C:/Users/...` 형식도 된다.

## 참고
- OpenSSH: ssh_config(5), `ssh -G`
