# docker exec 창을 닫아도 컨테이너 안 프로세스가 남는 이유

> 분류: 기타 · 최근 갱신: 2026-10-23

## 한 줄 요약
`ssh -t host "docker exec -it <컨테이너> tail -F <로그>"`로 띄운 창을 닫아도 컨테이너 안의 `tail`은 신호를 받지 않아 계속 남는다. `-t`를 빼고 `-i`만 쓴 뒤, 입력이 끊기면(EOF) 스스로 `tail`을 죽이는 작은 스크립트로 감싸면 해결된다.

## 내용
### 왜 남는가
1. 창을 닫으면 PC의 ssh가 끝나고 서버 sshd가 세션을 닫는다. 호스트에서 돌던 `docker exec` 클라이언트는 SIGHUP을 받고 죽는다.
2. 그런데 컨테이너 안의 `tail`은 그 클라이언트의 자식이 아니다. dockerd/containerd가 띄운 프로세스라 클라이언트가 죽어도 신호가 전달되지 않는다(Docker에서 오래전부터 알려진 동작).
3. `-t`로 만든 가상 터미널은 containerd-shim이 계속 붙잡고 있다. 그래서 `tail`이 출력을 써도 실패하지 않고, `-F`는 스스로 끝나지도 않아 계속 남는다.
- 호스트에서 바로 `ssh host "tail -F ..."`로 띄운 경우는 sshd가 세션을 닫을 때 `tail`도 SIGHUP을 받고 끝나므로 이 문제가 없다.
- `docker exec -it <컨테이너> bash`로 연 셸 창도 같은 이유로 남을 수 있다.
- 네트워크가 바뀌어 연결이 끊긴 경우에는 서버가 끊긴 걸 바로 알지 못해, 호스트 쪽 sshd와 `docker exec`도 한동안 남는다.

### 해결 방법
- `-it` 대신 `-i`만 쓴다. 입력이 가상 터미널이 아니라 일반 파이프로 연결되어, 바깥 연결이 끊기면 컨테이너 안에서 EOF를 받을 수 있다.
- 컨테이너 안에서 `tail`을 백그라운드로 돌리고, `cat`으로 입력이 끝나기를 기다렸다가 `kill`로 `tail`을 끈다. `cat`은 "연결이 아직 살아 있나"를 지켜보는 감시자 역할이다.
- 창용 ssh에 `-o ServerAliveInterval=30`을 붙이면 네트워크가 끊겼을 때 PC 쪽 ssh가 빨리 끝난다. 서버 쪽 정리는 sshd 설정(ClientAlive*)에 달려 있다.

## 예시
```bash
# 창을 닫거나 Ctrl+C를 누르면 컨테이너 안 tail도 함께 끝난다
ssh -o ServerAliveInterval=30 user@host \
  "docker exec -i <컨테이너> sh -c 'tail -n 200 -F \"\$1\" & t=\$!; cat >/dev/null; kill \$t' sh /path/to/app.log"

# 남아 있는 tail 확인
docker exec <컨테이너> ps -ef | grep 'tail -n 200 -F'
```
- `sh -c '<스크립트>' sh <경로>`: 첫 `sh`는 `$0` 자리 이름이고 경로가 `$1`이 된다.
- `$!`는 방금 백그라운드로 띄운 프로세스 번호다.

## 주의할 점
- `-t`를 빼면 터미널이 아니므로 색 표시 같은 터미널 기능이 빠질 수 있다. 로그 보기에는 문제없다.
- 명령을 Windows cmd 등 다른 셸의 큰따옴표 안에 넣으면 안쪽 큰따옴표가 잘릴 수 있다. 따옴표 없이 `$1`을 쓰면 경로에 공백이 없어야 한다.
- 이미 남아 있는 프로세스는 소유자를 확인한 뒤 정리한다. `ps` 출력 읽는 법은 같은 분류의 "ps STAT 값(Ss+)과 남아 있는 tail -F 프로세스" 노트를 참고한다.

## 참고
- Docker 문서: docker exec (-i, -t)
- man ssh_config (ServerAliveInterval), man sshd_config (ClientAliveInterval)
