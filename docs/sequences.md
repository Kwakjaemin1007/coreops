# 시퀀스 분석

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
