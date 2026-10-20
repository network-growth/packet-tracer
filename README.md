# Enterprise Network in Cisco Packet Tracer

본사(3계층) + 지사 2곳 + 듀얼 ISP로 구성한 기업망을 Packet Tracer 9.0으로 직접 구축하는 개인 프로젝트입니다.
매일 단계별로 설정하고, 장비 설정과 트러블슈팅 기록을 함께 남깁니다.

## 토폴로지 개요

| 구역 | 구성 |
|---|---|
| 인터넷 | ISP-A (AS100), ISP-B (AS200), INET-CORE (AS300), 공용 서버 |
| 본사 엣지 | HQ-EDGE1 / HQ-EDGE2 (2911) — eBGP 멀티호밍, NAT, GRE/IPsec |
| 본사 코어 | HQ-CORE1 / HQ-CORE2 (3650) — L3 EtherChannel |
| 건물 A / B | 분배 스위치 쌍(3560) + 액세스 스위치 3대씩(2960), HSRP |
| 데이터센터 | DC-SW1 / DC-SW2 — DHCP, DNS, AAA, Syslog/NTP 서버 |
| DMZ | ASA 5506-X — 웹 / 메일 / DNS 공개 서버 |
| 무선 | WLC-2504 + LAP 2대 (WPA2-Enterprise / PSK) |
| 지사1 | ISR4331 — GRE over IPsec, OSPF |
| 지사2 | 2911 + 3560 — Serial, EIGRP ↔ OSPF 재분배 |

장비 약 60대, 케이블 78개. 자세한 주소·포트 계획은 [docs/design.md](docs/design.md)를 참고하세요.

## 사용 기술

`VLAN` `802.1Q` `Rapid-PVST+` `LACP EtherChannel` `HSRPv2` `OSPF Multi-Area` `EIGRP` `Redistribution`
`eBGP / iBGP` `NAT/PAT` `GRE over IPsec` `ASA` `ACL` `DHCP Relay` `WLC` `RADIUS / TACACS+`
`Port-Security` `DHCP Snooping` `DAI` `SSH`

## 진행 현황

| 일차 | 주제 | 상태 | 기록 |
|---|---|---|---|
| Day 1 | 기본 설정, VLAN, 트렁크, EtherChannel, STP, 액세스 포트 | ✅ 완료 | [day1-L2.md](docs/day1-L2.md) |
| Day 2 | Routed 링크, L3 EtherChannel, SVI, HSRP, OSPF, DHCP | ⏳ | |
| Day 3 | BGP 멀티호밍, NAT | ⏳ | |
| Day 4 | 지사 연결 (GRE/IPsec, EIGRP 재분배) | ⏳ | |
| Day 5 | ASA DMZ, ACL | ⏳ | |
| Day 6 | 무선, AAA, NTP/Syslog | ⏳ | |
| Day 7 | L2 보안, 최종 검증 | ⏳ | |

## 저장소 구조
├── README.md
├── topology/
│ └── Network-Project.pkt # Packet Tracer 파일 (최신 상태)
├── docs/
│ ├── design.md # 설계서: IP / VLAN / 포트 연결표
│ └── day1-L2.md # 일차별 작업 기록 + 트러블슈팅
└── configs/
└── day1/ # 일차 종료 시점 장비별 running-config

## 파일 여는 법

- `topology/Network-Project.pkt` — Cisco Packet Tracer 9.0 이상에서 열기
- `configs/` — 장비별 설정 텍스트. 일차별 폴더를 비교하면 무엇이 추가됐는지 볼 수 있습니다.

## 참고

- 설정 파일의 비밀번호 해시는 `<removed>`로 가렸습니다.
- 학습 목적의 실습 환경이며, 공인 IP는 문서/예제용 대역을 사용했습니다.
