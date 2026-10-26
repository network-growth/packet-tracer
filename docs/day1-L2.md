# Day 1 — L2 구성

## 오늘 한 것
| 단계 | 내용 | 대상 장비 |
|---|---|---|
| 1-1 | 기본 설정 (hostname, SSH v2, 로컬 계정) | 라우터 7, L3 9, L2 10 |
| 1-2 | VLAN 생성 (부서별, 666 BLACKHOLE, 999 NATIVE) | 스위치 전체 |
| 1-3 | 802.1Q 트렁크, Native VLAN 999, allowed VLAN 제한 | 업링크 |
| 1-4 | L2 EtherChannel Po2 (LACP) | DIST-A/B 쌍, DC 쌍 |
| 1-5 | Rapid-PVST+, VLAN별 루트 브리지 분산 | DIST, DC |
| 1-6 | 액세스 포트 (PortFast, BPDU Guard), 미사용 포트 차단 | ACC, DIST, DC, 지사 |

## 트러블슈팅: Po2가 묶이지 않음
- 증상: (내가 본 것 — 예: 링크 주황/빨강, show etherchannel summary에 Po2(SD), 멤버 (s))
- 원인: (내 말로 — 멤버 포트에 channel-group을 먼저 넣어서 Po2가 Native VLAN 1로 생성됨)
- 해결: (내가 한 순서)
- 배운 점:

## 검증 결과
- show etherchannel summary
  <img width="869" height="696" alt="image" src="https://github.com/user-attachments/assets/b6ba1fb9-a3a5-4502-8d8f-a56f85f0736b" />

- show spanning-tree vlan 110
  <img width="872" height="693" alt="image" src="https://github.com/user-attachments/assets/fda9ba73-d463-4c22-a067-ac95f255c5b0" />

  <img width="864" height="692" alt="image" src="https://github.com/user-attachments/assets/ef6a3ab4-b3d2-4ff4-ac80-e8d45ccb0a50" />

- show vlan brief
  <img width="866" height="695" alt="image" src="https://github.com/user-attachments/assets/81ff774d-5df4-41ad-b6d6-d67643d68a51" />

## 남은 이슈 / 다음에 할 것
- BR2-L3SW Fa0/1 Native VLAN 불일치 (CDP 로그)
- Day 2: L3, HSRP, OSPF, DHCP
