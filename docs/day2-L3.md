# Day 2 — 본사 내부 L3 + OSPF + DHCP

## 오늘 한 것
| 단계 | 내용 | 대상 장비 |
|---|---|---|
| 2-1 | `ip routing`, Routed 링크(/30), Loopback0 | 엣지 2, 코어 2, 분배 4, DC 2 |
| 2-2 | 코어 간 L3 EtherChannel Po1 (LACP) | HQ-CORE1, HQ-CORE2 |
| 2-3 | SVI + HSRPv2 (VLAN별 Active 분산), DHCP 릴레이 | 분배 4, DC 2 |
| 2-3 | 액세스 스위치 관리 IP + default-gateway | ACC-A1~A3, ACC-B1~B3 |
| 2-4 | OSPF 멀티 에어리어 (Area 0 / 10 / 20 / 30) | 엣지, 코어, 분배, DC |
| 2-5 | DHCP 서버 풀 8개, 서버·프린터 고정 IP, PC DHCP | SRV-DHCP 외 |

## 설계 포인트
- **코어↔분배를 L3로:** STP 범위를 건물 안으로 제한하고, OSPF로 두 코어를 동시에 사용
- **HSRP Active = STP Root:** L2 경로와 게이트웨이가 같은 스위치라 트래픽이 우회하지 않음
- **HSRP version 2:** 그룹 번호를 VLAN 번호와 맞추기 위해 (299, 310 등 255 초과)
- **ABR 요약:** 코어에는 건물별 /16 한 줄만 (10.1.0.0/16, 10.2.0.0/16, 10.3.0.0/16)
- **Area 10 Totally Stubby:** 건물 A 안에는 기본 경로 하나만
- **DHCP 릴레이:** SRV-DHCP 한 대로 모든 VLAN에 주소 배포 (`ip helper-address`)

## 트러블슈팅

### 1. 3650에서 `ip address`가 안 들어감
- 증상: `% Invalid input`, 포트 이름 `gig0/1`도 인식 안 됨
- 원인: 3650은 포트 이름이 `Gi1/0/x` (3단). 또 스위치 포트는 기본이 L2라 IP를 못 가짐
- 해결: `interface gig1/0/x` → `no switchport` → `ip address`

### 2. `router ospf 1`에서 "IP routing not enabled"
- 원인: L3 스위치에 `ip routing`이 빠짐
- 해결: `ip routing` 먼저 입력 후 OSPF 설정
- 배운 점: `ip routing`이 없으면 OSPF뿐 아니라 VLAN 간 통신도 안 됨

### 3. HSRP VLAN 110만 양쪽 다 Active + Duplicate address
- 증상: DIST-A1 `show standby brief`에 VLAN 110이 안 보이거나 양쪽 Active,
  로그에 `%IP-4-DUPADDR: Duplicate address 10.1.10.1 on Vlan110`
- 원인: DIST-A1 VLAN 110에만 `standby version 2`가 빠져 v1로 동작 → v1/v2는 서로 Hello를 못 알아들음
- 단서: 로그의 가상 MAC — `0000.0C07.ACxx`는 v1, `0000.0C9F.Fxxx`는 v2
- 해결: 해당 SVI에서 `no standby 110` 후 v2로 재생성, SVI shutdown / no shutdown
- 배운 점: (내 말로)

### 4. OSPF network 명령 와일드카드 실수
- 실수: `network 10.0.0.0 0.0.0.0 area 0` → 아무 포트도 안 걸림
- 원인: `0.0.0.0`은 "정확히 이 주소 하나". 10.0.0.0을 가진 포트는 없음
- 정리: 포트 IP를 쓸 땐 `0.0.0.0`, 네트워크 주소를 쓸 땐 범위 와일드카드(`0.0.0.3`, `0.0.0.255`)

## 검증 결과
- `show etherchannel summary` (HQ-CORE1)
  
  <img width="869" height="693" alt="image" src="https://github.com/user-attachments/assets/64835576-78b5-4643-a2e7-7c92307b7dc5" />

- `show standby brief` (DIST-A1)

  <img width="873" height="693" alt="image" src="https://github.com/user-attachments/assets/32d4941d-5a4b-40f2-89d4-3b3d3654dd75" />

- `show ip ospf neighbor` (HQ-CORE1)
  
  <img width="871" height="693" alt="image" src="https://github.com/user-attachments/assets/be4db436-aa53-42b5-870e-171f2a985964" />

- `show ip route ospf` (HQ-CORE1)
  
  <img width="871" height="693" alt="image" src="https://github.com/user-attachments/assets/06d46d4b-4e2d-463d-9a89-6f6120687254" />

- PC-HR1 `ipconfig` → (받은 IP)
  
  <img width="866" height="695" alt="image" src="https://github.com/user-attachments/assets/a02ba584-50be-448d-b7e5-14fd41a52013" />

- PC-HR1 → PC-ENG1 / SRV-DHCP ping
  
  <img width="798" height="203" alt="image" src="https://github.com/user-attachments/assets/1d4e1167-20b1-4d79-b7a6-7af52c2f70ba" />

  <img width="795" height="196" alt="image" src="https://github.com/user-attachments/assets/1f9184c4-37ed-4e54-9e9a-fe364aa864fd" />

## 남은 이슈 / 다음에 할 것
- HQ-CORE1 Gi1/0/11 down — 반대편 ASA 미설정 (Day 5)
- BR2-L3SW Fa0/1 Native VLAN 불일치 (Day 4 전에 수정)
- Day 3: BGP 멀티호밍, NAT, 기본 경로 전파
