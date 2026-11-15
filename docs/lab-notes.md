# 실습 기록

## S0: 로컬 5G 코어 구동 (2026-11-16)

### 환경
- 실습 서버: Windows 데스크톱, WSL2 Ubuntu 24.04 (커널 6.18.58.1-microsoft-standard-WSL2)
- 개발 기기: MacBook Air M4 (코드·문서 작성)
- 사전 확인: SCTP 모듈 로드(`modprobe sctp`), `/dev/net/tun` 존재, Docker 컨테이너 amd64 이미지 동작

### 구성
- Open5GS 5G SA 코어 + UERANSIM(gNB, UE)을 docker-compose로 구동
- 사용한 레포: herlesupreeth/docker_open5gs (미리 빌드된 이미지 사용)
- 실행 순서: 코어(`sa-deploy.yaml`) → WebUI에서 가입자 등록 → gNB(`nr-gnb.yaml`) → UE(`nr-ue.yaml`)

### 결과
- UE 로그: PDU 세션 수립 성공, TUN 인터페이스 `uesimtun0`(192.168.100.2) 생성
- 데이터 경로 확인: UE의 `uesimtun0`을 통해 8.8.8.8로 ping, 3/3 응답, 손실 0%
- 지연: 평균 약 35.5ms (같은 PC에서 직접 8.8.8.8로 보낸 ping은 약 34ms)

```text
$ docker exec nr_ue ping -I uesimtun0 -c 3 8.8.8.8
3 packets transmitted, 3 received, 0% packet loss
rtt min/avg/max/mdev = 34.847/35.550/36.199/0.553 ms
```

### 확인된 경로
UE → gNB → (N3, GTP-U) → UPF → 인터넷
