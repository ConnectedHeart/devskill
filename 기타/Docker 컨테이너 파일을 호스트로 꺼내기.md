# Docker 컨테이너 파일을 호스트로 꺼내기

> 분류: 기타 · 최근 갱신: 2026-10-07

## 한 줄 요약
SFTP 클라이언트는 컨테이너 안을 직접 볼 수 없으므로, 서버에서 `docker cp`로 파일을 호스트 경로에 꺼낸 뒤 그 경로를 받는다.

## 내용
1. 서버에 SSH로 접속해 `docker ps`로 컨테이너 이름을 확인한다.
2. `docker cp`로 컨테이너 안의 파일(또는 폴더)을 호스트로 복사한다. 컨테이너가 중지 상태여도 된다.
3. SFTP 클라이언트(프로토콜 SFTP, 포트 22, SSH 계정)로 접속해 꺼낸 경로에서 내려받는다.
4. 받은 뒤 호스트에 꺼내 둔 사본을 지운다.

- 컨테이너 안 경로를 모르면 `docker exec -it <컨테이너> sh`로 들어가 찾는다.
- 볼륨으로 마운트된 경로라면 파일이 이미 호스트에 있다. `docker inspect <컨테이너>`의 `Mounts` 항목에서 `Source` 경로를 확인해 바로 받으면 된다.

## 예시
```bash
docker ps
docker cp <컨테이너>:/app/config/app.properties /tmp/
sudo chown <계정> /tmp/app.properties   # root 소유라 못 읽을 때
docker inspect <컨테이너> --format '{{ json .Mounts }}'
```

## 주의할 점
- `docker cp`는 `sudo`가 필요할 수 있고, 꺼낸 파일이 root 소유면 SFTP 계정으로 읽지 못한다.
- 설정 파일에는 접속 정보가 들어 있을 수 있으니 `/tmp`에 남겨 두지 않는다.

## 참고
- Docker CLI reference — docker cp, docker inspect
