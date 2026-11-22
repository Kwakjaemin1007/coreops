# 시퀀스 분석

## 전체 구조

```text
==========================================================================================
  [ 제어 평면 (Control Plane) : 신원 확인 · IP 할당 · 해제/통제 명령이 오가는 길 ]
==========================================================================================

                                      ┌──────────┐   ┌──────────┐   ┌──────────┐
                                      │   NRF    │   │   PCF    │   │   NSSF   │
                                      │(전화번호부)│   │(속도정책) │   │ (슬라이스) │
                                      └────┬─────┘   └────┬─────┘   └────┬─────┘
                                           │              │              │
   ┌──────────┐       N2 (SCTP 38412)      ▼              ▼              ▼
   │          │ ◀───────────────────▶ ┌──────────┐   ┌──────────┐   ┌──────────┐
   │   gNB    │                       │   AMF    │──▶│   AUSF   │──▶│ UDM / UDR│
   │ (기지국)  │                       │(정문 출입)│   │(신분인증) │   │(가입자 DB)│
   │          │                       └────┬─────┘   └──────────┘   └──────────┘
   └────┬─────┘                            │
        │                                  ▼
        │                             ┌──────────┐
        │                             │   SMF    │  (IP 배정 & 세션 개통/해제 지시)
        │                             └────┬─────┘
        │                                  │
        │                                  │  N4 (UDP 8805) : "길 열어줘 / 닫아줘!"
        │                                  ▼
========│==================================│==============================================
  [ 사용자 평면 (User Plane) : 실제 카톡 · 유튜브 패킷이 달리는 데이터 고속도로 ]
========│==================================│==============================================
        │                                  │
        │        N3 (UDP 2152 / GTP-U)     ▼
        └───────────────────────────▶ ┌──────────┐     TUN 장치     ┌──────────────┐
                                      │   UPF    │ ───────────────▶ │  외부 인터넷   │
  ┌──────────┐       무선 구간         │(물류터미널)│  (5G 포장 해제)  │ (Google 등)  │
  │    UE    │ ◀ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─▶ └──────────┘                  └──────────────┘
  │(스마트폰) │
  └──────────┘
```

## 1. 등록 (Registration)

### 예측 (캡처 전에 작성)
UE가 처음 켜졌을 때 메시지가 오가는 순서를 적는다.

1. UE → gNB: RRC Setup Request (기지국에 무선 연결 통로 개설 요청)
2. gNB → UE: RRC Setup (기지국이 무선 연결 설정 승인)
3. UE → gNB → AMF: Registration Request (NGAP Initial UE Message에 실려 AMF에 등록 요청, 임시 식별자 SUCI 전달)
4. AMF → AUSF → UDM: Nausf_UEAuthentication_Get Request (UDM에서 인증 벡터 조회 및 수신)
5. AMF → gNB → UE: Authentication Request (난수 RAND와 네트워크 인증 토큰 AUTN 전달)
6. UE → gNB → AMF: Authentication Response (유심의 Ki, OP로 연산한 응답값 RES* 전송, 상호 인증 완료)
7. AMF → gNB → UE: Security Mode Command (이후 NAS 메시지의 암호화·무결성 보호 알고리즘 지정)
8. UE → gNB → AMF: Security Mode Complete (보안 모드 적용 완료 응답)
9. AMF → gNB → UE: Registration Accept (최종 등록 승인, 임시 식별자 5G-GUTI 발급)
10. UE → gNB → AMF: Registration Complete (등록 승인 수신 확인)

### 질문 (내가 답할 것)
- 인증은 누가 누구에게 무엇을 보내서 하는가?
  - 코어망(AUSF/AMF)이 gNB를 거쳐 UE에게 Authentication Request로 난수(RAND)와 인증 토큰(AUTN)을 보낸다. UE는 유심(Ki, OP)으로 연산한 응답값(RES*)을 Authentication Response에 담아 AMF → AUSF로 회신하고, 이로써 단말과 네트워크가 서로를 확인(상호 인증)한다. (인증 벡터는 AUSF/UDM이 만들고, AMF는 이를 UE에 전달한다.)
- Security Mode Command는 왜 필요한가?
  - 인증이 끝나기 전까지의 제어 메시지는 보호되지 않는다. 이후 오가는 단말 성능, 슬라이스 정보, PDU Session 요청 같은 민감한 제어 정보가 도청·변조되지 않도록 암호화와 무결성 보호를 시작하는 절차다.
- AMF가 gNB에 보내는 "Registration Accept"는 어떤 NGAP 메시지에 실려 가는가?
  - InitialContextSetupRequest에 실려 간다. Registration Accept는 UE와 AMF만 해석하는 NAS 메시지라서, AMF가 gNB에게 무선 자원과 보안 키 설정(Context 생성)을 지시하는 N2 메시지 안에 캡슐화해서 보낸다.

### 결과 (캡처 결과)
- 1~5번 예측 순서가 일치했다 (등록 요청 → 인증 → 보안 모드)
- Registration Accept는 InitialContextSetupRequest(NGAP)에 실려 간다
- 예측 1~2번(RRC)은 UE와 gNB 사이 무선 구간이라 코어 쪽 캡처에는 나오지 않는다
- 예측에 없던 것: UEContextRelease가 보안 모드 직후에 나온다 (원인은 확인 중)
- 등록 직후 PDU 세션 요청이 이어지고, 마지막에 PDUSessionResourceSetup 요청/응답이 나온다

## 2. PDU 세션 수립

### 예측
(작성 예정)

### 결과 (캡처 결과)
- Frame 1186: SMF → AMF로 HTTP/2 메시지가 오간다. JSON 안에 N1(NAS-SM)과 N2(PDU_RES_SETUP_REQ) 조각이 들어 있다
- 이후 gNB↔AMF 구간에서 PDUSessionResourceSetupRequest/Response가 나온다 (Frame 1193, 1206)

## 3. 해제 (Deregistration)

### 예측
1. UE → AMF: Deregistration request (UE가 gNB를 거쳐 AMF에 접속 해제 요청)
2. AMF → SMF / gNB:
   - AMF → SMF: 활성 PDU Session 해제 요청. SMF는 N4로 UPF의 전달 규칙(PDR/FAR) 삭제를 지시
   - AMF → gNB: UE Context Release Command (N2로 단말의 무선·연결 자원 정리 지시)
3. gNB / SMF → AMF:
   - gNB → AMF: UE Context Release Complete (자원 정리 완료 보고)
   - SMF → AMF: 세션 해제 완료 응답
   - 일반 해제라면 AMF → UE Deregistration accept도 함께 확인한다

### 확인할 것 (캡처 후)
- Deregistration accept가 Release Command 앞인가, 뒤인가, 아예 없는가
- AMF→SMF 세션 해제는 http2 필터에서 어떻게 보이는가
- UE를 compose down으로 끈 경우 정상 해제 흐름이 맞게 나오는가

### 가설 (캡처 전에 일반 지식으로 쓴 것, 캡처 후 실제 결과로 바꾼다)
- 일반 해제(Normal deregistration)에서는 보통 Deregistration accept가 UEContextReleaseCommand보다 앞에 나오고, 전원 끄기(Switch off) 유형이면 accept가 없는 경우도 있다.
- http2 필터에서는 AMF와 SMF 사이의 nsmf-pdusession 계열 요청과 200 OK / 204 No Content 응답으로 보일 것이다. SMF는 이를 바탕으로 N4로 UPF 규칙을 삭제한다.
- compose down은 컨테이너에 종료 신호를 보낼 뿐이라, UE가 Deregistration request를 보낸다는 보장이 없다. 보내지 않으면 코어가 타이머 만료로 정리할 수도 있다.

### 결과 (캡처 결과)
캡처: UERANSIM `nr-cli`의 `deregister normal` 실행 (dereg-normal.pcap)

**확인된 것 (패킷에서 직접 확인)**
- gNB→AMF UplinkNASTransport (No.249) 후, AMF→gNB DownlinkNASTransport (No.433)가 나온다
- AMF→gNB UEContextReleaseCommand (No.435), Cause: Nas=deregister
- gNB→AMF UEContextReleaseComplete (No.441)
- 예측 2~3번(Release Command → Release Complete)과 일치했다
- 해제 후 UE가 자동으로 다시 등록한다 (No.507~)

**추정 (패킷만으로 확인 불가)**
- 249번은 Deregistration request, 433번은 Deregistration accept로 보인다. 등록 때 Security Mode 이후의 NAS 메시지는 암호화되어 Wireshark가 메시지 종류를 풀지 못한다
- 따라서 "accept가 Release Command보다 앞"은 순서상 그렇게 보일 뿐 단정하지 않는다

**배운 것**
- `docker compose down`으로 끈 UE는 해제 메시지 없이 종료되어, 코어에 해제 흔적이 남지 않았다 (SCTP/PFCP는 Heartbeat만 있었다)
- 정상 해제는 `nr-cli <imsi> --exec "deregister normal"`로 일으킨다

**N4 확인 (PFCP)**
- SMF→UPF PFCP Session Deletion Request (No.263), UPF의 Response (No.265)가 UEContextReleaseCommand (No.435)보다 먼저 나온다
- 순서: UE 해제 요청 → 세션 삭제(N4) → gNB 컨텍스트 해제
- 이전 `compose down` 캡처에는 Session Deletion이 없었다
- 남은 확인: AMF↔SMF 구간(http2)