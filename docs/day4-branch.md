# Day 4 — 지사 연결 (GRE over IPsec / Serial + EIGRP)

## 오늘 한 것
| 단계 | 내용 | 대상 장비 |
|---|---|---|
| 4-1 | BR1-R Router-on-a-stick (VLAN 10 / 99 / Native 999), 지사1 DHCP, BR1-SW 트렁크 | BR1-R, BR1-SW |
| 4-2 | GRE 터널 HQ-EDGE1 ↔ BR1-R (172.16.1.0/30, ISP-A 경유) | HQ-EDGE1, BR1-R |
| 4-3 | IPsec으로 GRE 암호화 (AES-256, SHA, PSK, DH group 5) | HQ-EDGE1, BR1-R |
| 4-4 | 터널 위 OSPF (Area 0), 지사1 LAN Area 11 요약, 지사1 로컬 인터넷 NAT | HQ-EDGE1, BR1-R |
| 4-5 | Serial 전용선 HQ-EDGE2 ↔ BR2-R (DCE: HQ-EDGE2), BR2-L3SW SVI/DHCP | HQ-EDGE2, BR2-R, BR2-L3SW, BR2-ACC1 |
| 4-6 | 지사2 EIGRP AS 100, HQ-EDGE2에서 OSPF ↔ EIGRP 상호 재분배 | HQ-EDGE2, BR2-R, BR2-L3SW |

## 구성 요약
| 지사 | 본사 연결 | 라우팅 | 인터넷 | DHCP |
|---|---|---|---|---|
| 지사1 (10.11.0.0/16) | GRE over IPsec (인터넷 경유) | OSPF Area 0 (터널) + Area 11 | 로컬 브레이크아웃 (BR1-R NAT → 100.0.2.2) | BR1-R |
| 지사2 (10.12.0.0/16) | Serial 전용선 | EIGRP AS 100 + 재분배 | 센트럴 브레이크아웃 (본사 HQ-EDGE2 NAT) | BR2-L3SW |

## 설계 포인트
- **GRE:** ISP는 사설 주소를 모르므로 사설 패킷을 공인 주소로 감싸 전달 (가상 직결선)
- **IPsec:** GRE는 암호화가 없어 ACL로 GRE 패킷만 골라 암호화. 일반 인터넷 트래픽은 암호화 대상 아님
- **재귀 라우팅 방지:** 터널 바깥 주소(100.x)는 OSPF에 넣지 않음
- **AD 우선순위:** 지사1 기본 경로 — 정적(AD 1)이 OSPF(AD 110)보다 우선 → 인터넷은 로컬, 본사행은 구체 경로로 터널
- **재분배:** OSPF → EIGRP는 metric 5개 필수, EIGRP → OSPF는 `subnets` 필수, 양방향 모두 필요
- **no auto-summary:** 본사·지사 모두 10.x라 클래스 단위 자동 요약 시 경로 충돌

## 트러블슈팅

### 1. 보안 라이선스(securityk9) 활성화 실패
- 증상: `crypto` 명령 미동작. `license boot module c2900 technology-package securityk9` 입력 후
  저장·재부팅해도 `show version`의 security가 `disable`, Next reboot도 `None`
- 시도: 재부팅, 전원 재시작, `license accept end user agreement`(PT 미지원 명령)
- 확인: 새 2911에서 같은 명령 입력 시 EULA(yes) 프롬프트가 나오고 정상 활성화됨
- 원인: (내 결론 — 예: 기존 HQ-EDGE1 장비의 라이선스 상태 문제)
- 해결: (내가 한 방법 — 예: 새 라우터로 교체 후 설정 Merge)
- 배운 점: 라이선스 상태는 `show version`의 Current / Next reboot 칸으로 확인

### 2. crypto map 생성 시 안내 메시지
- 메시지: `% NOTE: This new crypto map will remain disabled until a peer and a valid access list have been configured.`
- 의미: 오류가 아니라 `set peer`, `match address` 입력 전이라 비활성 상태라는 안내

## 검증 결과
- HQ-EDGE1 `show ip ospf neighbor` → (BR1-R 10.255.0.41 Tunnel0 FULL)

  <img width="761" height="784" alt="image" src="https://github.com/user-attachments/assets/0f2908d5-24df-4485-851f-b3335f0ac39f" />

- HQ-EDGE1 `show crypto isakmp sa` / `show crypto ipsec sa` → (QM_IDLE, encaps/decaps 수)

  <img width="707" height="240" alt="image" src="https://github.com/user-attachments/assets/42c4da0a-e451-4a5b-87ca-6ad15b59519d" />

- HQ-EDGE2 `show ip eigrp neighbors` → (BR2-R)
  
  <img width="1037" height="780" alt="image" src="https://github.com/user-attachments/assets/2038a8e9-d1e5-49f1-ac0d-07f01d9440c5" />
  
- HQ-CORE1 `show ip route` → (O IA 10.11.0.0/16, O E2 10.12.x.x)

  <img width="1037" height="783" alt="image" src="https://github.com/user-attachments/assets/fc46ae6c-e538-491a-8c9d-2aa27867f4a4" />

- BR1-PC1 / BR2-PC1 `ipconfig` → (받은 IP)
  
  <img width="1032" height="772" alt="image" src="https://github.com/user-attachments/assets/df7f8653-c175-4835-8c44-195f5f2fff84" />

  <img width="1032" height="781" alt="image" src="https://github.com/user-attachments/assets/47e3dfb5-ff93-41ed-a377-99b46decba02" />


- BR1-PC1 → PC-HR1, BR2-PC1 → PC-HR1, BR1-PC1 ↔ BR2-PC1 ping → (결과)

  <img width="986" height="307" alt="image" src="https://github.com/user-attachments/assets/af0a18be-3c7d-4757-b946-96987d021a59" />

  <img width="703" height="299" alt="image" src="https://github.com/user-attachments/assets/fa4dfa39-4acb-48a4-ac39-debbe6eb1418" />

  <img width="687" height="279" alt="image" src="https://github.com/user-attachments/assets/1136669e-5a4b-4093-b115-3b7ebe26a0c2" />

- BR1-PC1 / BR2-PC1 → 8.8.8.8 ping → (결과)

  <img width="687" height="288" alt="image" src="https://github.com/user-attachments/assets/bad8491b-7108-46f1-8663-95a7d18c522c" />

  <img width="691" height="207" alt="image" src="https://github.com/user-attachments/assets/7c25703e-4446-4f0f-b1a0-862ff58dc64b" />

## 남은 이슈 / 다음에 할 것
- 지사2는 본사 회선(HQ-EDGE2 ISP-B)에 인터넷을 의존 → 회선 장애 시 지사2 인터넷 불가
- Day 5: ASA 방화벽 DMZ, ACL (게스트·HR/FIN 차단)
