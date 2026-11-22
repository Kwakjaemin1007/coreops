# 시퀀스 분석
==========================================================================================
  [ 제어 평면 (Control Plane) : 신원 확인 · IP 할당 · 해제/통제 명령이 오가는 길 ]
==========================================================================================

                                      ┌──────────┐   ┌──────────┐   ┌──────────┐
                                      │   NRF    │   │   PCF    │   │   NSSF   │
                                      │(전화번호부)│    │(속도정책)  │   │ (슬라이스) │
                                      └────┬─────┘   └────┬─────┘   └────┬─────┘
                                           │              │              │
   ┌──────────┐       N2 (SCTP 38412)      ▼              ▼              ▼
   │          │ ◀───────────────────▶ ┌──────────┐   ┌──────────┐   ┌──────────┐
   │   gNB    │                       │   AMF    │──▶│   AUSF   │──▶│ UDM / UDR│
   │ (기지국)  │                        │(정문 출입) │   │ (신분인증) │   │ (가입자 DB)│
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
                                      │   UPF    │ ───────────────▶│  외부 인터넷    │
  ┌──────────┐       무선 구간           │(물류터미널)│  (5G 포장 해제)    │. (Google. 등)│
  │    UE    │ ◀ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─▶ └──────────┘                  └─────────────┘
  │(스마트폰)  │
  └──────────┘

## 1. 등록 (Registration)

### 예측 (캡처 전에 작성)
UE가 처음 켜졌을 때 메시지가 오가는 순서를 적는다.

1. UE → gNB: RRC Setup Request (기지국에 무선 연결 통로 개설 요청)
2. gNB → UE: RRC Setup (기지국이 무선 연결 설정 승인)
3. UE → gNB → AMF: Registration Request (기지국의 NGAP Initial UE Message에 실려 코어 정문 AMF에 망 등록 요청 및 임시 식별자 SUCI 전달)
4. AMF → AUSF → UDM: Nausf_UEAuthentication_Get Request (가입자 금고 UDM에서 인증 벡터 조회 및 수신)
5. AMF → gNB → UE: Authentication Request (단말 신원 확인을 위한 난수 RAND 및 네트워크 인증 토큰 AUTN 전달)
6. UE → gNB → AMF: Authentication Response (단말 유심의 Ki, OP 키로 연산한 응답값 RES*를 전송하여 상호 인증 완료)
7. AMF → gNB → UE: Security Mode Command (이후 오갈 NAS 메시지의 암호화 및 무결성 보호 알고리즘 지정)
8. UE → gNB → AMF: Security Mode Complete (보안 모드 적용 완료 응답)
9. AMF → gNB → UE: Registration Accept (최종 등록 승인 및 임시 식별자 5G-GUTI 발급)
10. UE → gNB → AMF: Registration Complete (등록 승인 수신 완료 확인)
### 질문 (내가 답할 것)
- 인증은 누가 누구에게 무엇을 보내서 하는가?
 -> 코어망(AUSF/AMF)이 기지국(gNB)을 거쳐 단말(UE)에게 Authentication Request 메시지로 난수(RAND)와 네트워크 인증 토큰(AUTN)을 보냅니다. 그리고 단말(UE)은 유심(Ki, OP)으로 연산한 응답값(RES*)을 Authentication Response 메시지에 담아 코어망(AMF → AUSF)으로 회신함으로써 단말과 네트워크가 서로 정당한 상대임을 확인(상호 인증)합니다.
- Security Mode Command는 왜 필요한가?
 ->인증(Authentication) 직전까지는 모든 제어 메시지가 암호화되지 않은 평문(Plain text)으로 전송됩니다. 이후 교환될 단말 성능, 네트워크 슬라이스 정보, 인터넷 연결(PDU Session) 요청 등의 민감한 제어 정보가 중간에 도청(암호화)되거나 위·변조(무결성 보호)되는 것을 막기 위해 보안 채널을 개시하는 절차입니다.
- AMF가 gNB에 보내는 "Registration Accept"는 어떤 NGAP 메시지에 실려 가는가?
 ->Initial Context Setup Request 메시지에 실려 갑니다.그리고 Registration Accept는 단말(UE)과 코어(AMF)만 해석하는 NAS 메시지이므로, AMF가 기지국(gNB)에게 단말의 무선 자원 및 보안 키 설정(Context 생성)을 지시하는 N2 전용 메시지(Initial Context Setup Request) 안에 단말용 Registration Accept를 캡슐화해 함께 전달합니다.

### 결과 
- 1~5번 예측 순서가 일치했다 (등록 요청 → 인증 → 보안 모드)
- Registration Accept는 InitialContextSetupRequest(NGAP)에 실려 간다
- 예측에 없던 것: UEContextRelease가 보안 모드 직후에 나온다 (원인은 확인 중)
- 등록 직후 PDU 세션 요청이 이어지고, 마지막에 PDUSessionResourceSetup 요청/응답이 나온다

## 3. 해제 (Deregistration)

### 예측
## 3. 해제 (Deregistration)

### 예측
1. **UE → AMF:** `Deregistration request` (단말이 gNB를 거쳐 코어 정문 AMF에 5G 접속 해제 요청 전달)
2. **AMF → SMF / gNB:**
   - **AMF → SMF:** 활성화된 데이터 세션(`PDU Session`) 해제 요청 → SMF가 `N4`를 통해 UPF에 전달 규칙(`PDR/FAR`) 삭제 지시
   - **AMF → gNB:** `UE Context Release Command` (`N2`를 통해 기지국에 단말 무선/연결 자원 정리 지시)
3. **gNB / SMF → AMF:**
   - **gNB → AMF:** `UE Context Release Complete` (기지국이 단말 자원 정리를 마쳤음을 확인 보고)
   - **SMF → AMF:** 세션 해제 완료 응답 전달 *(전원 끄기 외 일반 해제의 경우 AMF → UE `Deregistration accept` 흐름도 함께 확인)*
### 확인할 것 
- Deregistration accept가 Release Command 앞인가, 뒤인가, 아예 없는가
- AMF→SMF 세션 해제는 http2 필터에서 어떻게 보이는가
- UE를 compose down으로 끈 경우 정상 해제 흐름이 맞게 나오는가

- **Deregistration accept가 Release Command 앞인가, 뒤인가, 아예 없는가**
  - 일반 해제(`Normal deregistration`)에서는 보통 **`Deregistration accept`가 `UEContextReleaseCommand`보다 앞**에 나오고, 단말 전원 끄기(`Switch off`) 유형이면 **아예 없는 경우**도 있다. Wireshark(`nas-5gs || ngap`)에서 프레임 번호(`No.`) 순서로 직접 확인함.
- **AMF→SMF 세션 해제는 http2 필터에서 어떻게 보이는가**
  - `http2` 필터에서 `AMF`와 `SMF` IP 간 세션 관리 서비스(`nsmf-pdusession` 계열 경로의 해제 요청과 `200 OK` / `204 No Content` 응답)로 보임. `SMF`는 이를 바탕으로 `N4`를 통해 `UPF` 규칙을 삭제함.
- **UE를 compose down으로 끈 경우 정상 해제 흐름이 맞게 나오는가**
  - `compose down`으로 프로세스가 정상 종료 신호를 받고 내려가면 `Deregistration request` $\rightarrow$ 세션/문맥 정리 $\rightarrow$ `UEContextReleaseCommand` / `Complete` 흐름이 잡힘. 강제 종료(`docker kill`)처럼 신호 없이 끊기면 `Deregistration request`가 빠지거나 원인 코드가 달라질 수 있음.
