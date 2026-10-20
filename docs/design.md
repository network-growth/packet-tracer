# 대규모 기업망 프로젝트 — Packet Tracer 9.0

구성: 본사(3계층: Core / Distribution / Access, 건물 2동 + 데이터센터 + DMZ) + 지사 2곳 + ISP 2개(멀티호밍) + 인터넷
규모: 네트워크 장비 약 30대, 엔드 장비 약 30대, 케이블 78개
※ 이 문서는 "무엇을 만들지"만 정의합니다. 설정 방법은 직접 해결하세요.

---

## 1. 전체 구조

```
                    [PUBLIC-DNS 8.8.8.8] [PUBLIC-WEB] [REMOTE-PC]
                                   \        |        /
                                       [INET-SW]
                                           |
                                      [INET-CORE]  AS300
                                           |
          AS100 [ISP-A] ------------------ [ISP-B] AS200
               /      \                       |
     [BR1-R]--'        \                      |
   (지사1, GRE/IPsec)    \                     |
          |          [HQ-EDGE1] ~~iBGP~~ [HQ-EDGE2]------Serial------[BR2-R] (지사2, EIGRP)
     [BR1-SW]             |  \          /  |                            |
                          |    \      /    |                       [BR2-L3SW]
                          |      \  /      |                            |
                          |      /  \      |                       [BR2-ACC1]
                     [HQ-CORE1]=====Po1=====[HQ-CORE2]     ← 모든 하위 장비와 풀메시(Routed)
                      |    |    \             /    |
              [HQ-FW] |    |     \           /     |
               (ASA)  |    |      \         /      |
                 |    |    |       \       /       |
            [DMZ-SW]  |    |        \     /        |
                      |    |         \   /         |
        ┌─────────────┘    └───────┐  \ /  ┌───────┘
   [DIST-A1]==[DIST-A2]   [DIST-B1]==[DIST-B2]   [DC-SW1]==[DC-SW2]
     건물 A (Area 10)        건물 B (Area 20)        데이터센터 (Area 30)
   ACC-A1 ACC-A2 ACC-A3    ACC-B1 ACC-B2 ACC-B3      서버 4대 + NOC
                                         └─ WLC + LAP 2대 (무선)
```

## 2. 장비 목록

| 구분 | 장비명 | PT 모델 | 추가 모듈 / 주의 |
|---|---|---|---|
| 인터넷 | ISP-A, ISP-B, INET-CORE | 2911 | |
| 인터넷 | INET-SW | 2960-24TT | |
| 본사 엣지 | HQ-EDGE1 | 2911 | IPsec 위해 securityk9 라이선스 |
| 본사 엣지 | HQ-EDGE2 | 2911 | **HWIC-2T** (전원 끄고 장착) |
| 본사 코어 | HQ-CORE1, HQ-CORE2 | 3650-24PS | **AC-POWER-SUPPLY** 장착해야 켜짐 |
| 방화벽 | HQ-FW | ASA 5506-X | |
| 건물 A | DIST-A1, DIST-A2 | 3560-24PS | |
| 건물 A | ACC-A1, ACC-A2, ACC-A3 | 2960-24TT | |
| 건물 B | DIST-B1, DIST-B2 | 3560-24PS | |
| 건물 B | ACC-B1, ACC-B2, ACC-B3 | 2960-24TT | |
| 무선 | WLC-B | WLC-2504 | |
| 무선 | LAP-1, LAP-2 | LAP-PT (또는 3702i) | 전원 어댑터 장착 (2960은 PoE 아님) |
| DC | DC-SW1, DC-SW2 | 3560-24PS | |
| DMZ | DMZ-SW | 2960-24TT | |
| 지사1 | BR1-R | ISR4331 | IPsec 라이선스 |
| 지사1 | BR1-SW | 2960-24TT | |
| 지사2 | BR2-R | 2911 | **HWIC-2T** |
| 지사2 | BR2-L3SW | 3560-24PS | |
| 지사2 | BR2-ACC1 | 2960-24TT | |

### 엔드 장비

| 위치 | 장비 | 종류 |
|---|---|---|
| 건물 A | PC-HR1, PC-HR2, PC-FIN1, PC-FIN2, PC-SALES1, PC-SALES2 | PC |
| 건물 A | PRINTER-A | Printer |
| 건물 B | PC-ENG1, PC-ENG2, PC-DEV1, PC-DEV2 | PC |
| 건물 B (무선) | LAPTOP-CORP1, LAPTOP-GUEST1 | Laptop (NIC를 **WPC300N**으로 교체) |
| 건물 B (무선) | PHONE-1 | Smartphone |
| DC | SRV-DHCP, SRV-DNS, SRV-AAA, SRV-LOG-NTP | Server |
| DC | NOC-ADMIN | PC |
| DMZ | DMZ-WEB, DMZ-MAIL, DMZ-DNS | Server |
| 인터넷 | PUBLIC-DNS, PUBLIC-WEB | Server |
| 인터넷 | REMOTE-PC | PC |
| 지사1 | BR1-PC1, BR1-PC2 | PC |
| 지사2 | BR2-PC1, BR2-PC2 | PC |
| 지사2 | BR2-PRN1 | Printer |

---

## 3. 주소 체계 규칙

- 본사 건물 A = `10.1.0.0/16`, 건물 B = `10.2.0.0/16`, DC = `10.3.0.0/16`
- 지사1 = `10.11.0.0/16`, 지사2 = `10.12.0.0/16`
- **VLAN 번호 = [건물번호][세 번째 옥텟]** → 예: VLAN 120 = 10.1.**20**.0/24, VLAN 230 = 10.2.**30**.0/24
- 게이트웨이(HSRP VIP) = `.1`, 1번 장비 = `.2`, 2번 장비 = `.3`, DHCP 범위 = `.11 ~ .254`
- 장비 간 링크 = `10.0.0.0/24`를 /30으로 쪼개서 사용, 표의 **A측이 낮은 IP**
- 루프백 = `10.255.0.x/32` (OSPF/BGP Router-ID)

## 4. VLAN / 서브넷

### 건물 A (DIST-A1/A2가 게이트웨이, OSPF Area 10)

| VLAN | 이름 | 서브넷 | VIP | DIST-A1 | DIST-A2 | HSRP Active / STP Root |
|---|---|---|---|---|---|---|
| 110 | HR | 10.1.10.0/24 | .1 | .2 | .3 | DIST-A1 |
| 120 | FIN | 10.1.20.0/24 | .1 | .2 | .3 | DIST-A2 |
| 130 | SALES | 10.1.30.0/24 | .1 | .2 | .3 | DIST-A1 |
| 199 | MGMT-A | 10.1.99.0/24 | .1 | .2 | .3 | DIST-A2 |

### 건물 B (DIST-B1/B2, OSPF Area 20)

| VLAN | 이름 | 서브넷 | VIP | DIST-B1 | DIST-B2 | Active / Root |
|---|---|---|---|---|---|---|
| 210 | ENG | 10.2.10.0/24 | .1 | .2 | .3 | DIST-B1 |
| 220 | DEV | 10.2.20.0/24 | .1 | .2 | .3 | DIST-B2 |
| 230 | WIFI-CORP | 10.2.30.0/24 | .1 | .2 | .3 | DIST-B1 |
| 240 | WIFI-GUEST | 10.2.40.0/24 | .1 | .2 | .3 | DIST-B2 |
| 250 | AP-MGMT | 10.2.50.0/24 | .1 | .2 | .3 | DIST-B1 |
| 299 | MGMT-B | 10.2.99.0/24 | .1 | .2 | .3 | DIST-B2 |

### 데이터센터 (DC-SW1/SW2, OSPF Area 30)

| VLAN | 이름 | 서브넷 | VIP | DC-SW1 | DC-SW2 | Active / Root |
|---|---|---|---|---|---|---|
| 310 | SERVERS | 10.3.10.0/24 | .1 | .2 | .3 | DC-SW1 |
| 320 | INFRA (AAA/로그) | 10.3.20.0/24 | .1 | .2 | .3 | DC-SW2 |
| 399 | NOC-MGMT | 10.3.99.0/24 | .1 | .2 | .3 | DC-SW1 |

### 지사

| 지사 | VLAN | 이름 | 서브넷 | 게이트웨이 |
|---|---|---|---|---|
| 지사1 | 10 | BR1-USERS | 10.11.10.0/24 | BR1-R 서브인터페이스 .1 |
| 지사1 | 99 | BR1-MGMT | 10.11.99.0/24 | BR1-R 서브인터페이스 .1 |
| 지사2 | 10 | BR2-USERS | 10.12.10.0/24 | BR2-L3SW SVI .1 |
| 지사2 | 20 | BR2-PRINTERS | 10.12.20.0/24 | BR2-L3SW SVI .1 |
| 지사2 | 99 | BR2-MGMT | 10.12.99.0/24 | BR2-L3SW SVI .1 |

### 공통

| VLAN | 용도 |
|---|---|
| 999 | Native VLAN (모든 트렁크) |
| 666 | BLACKHOLE (안 쓰는 포트 전부, shutdown) |

---

## 5. 링크 주소 (Point-to-Point)

### 본사 내부 (10.0.0.0/24, /30)

| # | 서브넷 | A측 (낮은 IP) | B측 |
|---|---|---|---|
| 1 | 10.0.0.0/30 | HQ-EDGE1 Gi0/1 | HQ-CORE1 Gi1/0/1 |
| 2 | 10.0.0.4/30 | HQ-EDGE1 Gi0/2 | HQ-CORE2 Gi1/0/1 |
| 3 | 10.0.0.8/30 | HQ-EDGE2 Gi0/1 | HQ-CORE1 Gi1/0/2 |
| 4 | 10.0.0.12/30 | HQ-EDGE2 Gi0/2 | HQ-CORE2 Gi1/0/2 |
| 5 | 10.0.0.16/30 | HQ-CORE1 Po1 | HQ-CORE2 Po1 (L3 EtherChannel) |
| 6 | 10.0.0.20/30 | HQ-CORE1 Gi1/0/5 | DIST-A1 Gi0/1 |
| 7 | 10.0.0.24/30 | HQ-CORE1 Gi1/0/6 | DIST-A2 Gi0/1 |
| 8 | 10.0.0.28/30 | HQ-CORE1 Gi1/0/7 | DIST-B1 Gi0/1 |
| 9 | 10.0.0.32/30 | HQ-CORE1 Gi1/0/8 | DIST-B2 Gi0/1 |
| 10 | 10.0.0.36/30 | HQ-CORE1 Gi1/0/9 | DC-SW1 Gi0/1 |
| 11 | 10.0.0.40/30 | HQ-CORE1 Gi1/0/10 | DC-SW2 Gi0/1 |
| 12 | 10.0.0.44/30 | HQ-CORE2 Gi1/0/5 | DIST-A1 Gi0/2 |
| 13 | 10.0.0.48/30 | HQ-CORE2 Gi1/0/6 | DIST-A2 Gi0/2 |
| 14 | 10.0.0.52/30 | HQ-CORE2 Gi1/0/7 | DIST-B1 Gi0/2 |
| 15 | 10.0.0.56/30 | HQ-CORE2 Gi1/0/8 | DIST-B2 Gi0/2 |
| 16 | 10.0.0.60/30 | HQ-CORE2 Gi1/0/9 | DC-SW1 Gi0/2 |
| 17 | 10.0.0.64/30 | HQ-CORE2 Gi1/0/10 | DC-SW2 Gi0/2 |
| 18 | 10.0.0.68/30 | HQ-CORE1 Gi1/0/11 | HQ-FW Gi1/1 (inside) |

### WAN / 터널

| 구간 | 서브넷 | A측 | B측 |
|---|---|---|---|
| GRE over IPsec (HQ↔지사1) | 172.16.1.0/30 | HQ-EDGE1 Tunnel0 .1 | BR1-R Tunnel0 .2 |
| Serial (HQ↔지사2) | 172.16.2.0/30 | HQ-EDGE2 Se0/3/0 .1 (**DCE**) | BR2-R Se0/3/0 .2 |
| 지사2 내부 | 10.12.0.0/30 | BR2-R Gi0/0 .1 | BR2-L3SW Gi0/1 .2 |

### 공인망

| 구간 | 서브넷 | A측 | B측 |
|---|---|---|---|
| ISP-A ↔ HQ-EDGE1 | 100.0.1.0/30 | ISP-A Gi0/0 .1 | HQ-EDGE1 Gi0/0 .2 |
| ISP-A ↔ BR1-R | 100.0.2.0/30 | ISP-A Gi0/2 .1 | BR1-R Gi0/0/0 .2 |
| ISP-A ↔ ISP-B | 150.0.0.0/30 | ISP-A Gi0/1 .1 | ISP-B Gi0/2 .2 |
| ISP-B ↔ HQ-EDGE2 | 200.0.1.0/30 | ISP-B Gi0/0 .1 | HQ-EDGE2 Gi0/0 .2 |
| ISP-B ↔ INET-CORE | 200.0.2.0/30 | ISP-B Gi0/1 .1 | INET-CORE Gi0/0 .2 |
| 인터넷 서버망 | 8.8.8.0/24 | INET-CORE Gi0/2 .1 | 서버들 |

### 본사 공인 대역 203.0.113.0/24 (BGP로 광고)

| 범위 | 용도 |
|---|---|
| 203.0.113.0/26 | HQ-EDGE1 PAT 풀 |
| 203.0.113.64/26 | HQ-EDGE2 PAT 풀 |
| 203.0.113.192/27 | DMZ 서버 (공인 IP 직접 사용, NAT 없음) |

### 루프백 / Router-ID

| 장비 | Loopback0 | 장비 | Loopback0 |
|---|---|---|---|
| HQ-EDGE1 | 10.255.0.1 | DIST-B1 | 10.255.0.23 |
| HQ-EDGE2 | 10.255.0.2 | DIST-B2 | 10.255.0.24 |
| HQ-CORE1 | 10.255.0.11 | DC-SW1 | 10.255.0.31 |
| HQ-CORE2 | 10.255.0.12 | DC-SW2 | 10.255.0.32 |
| DIST-A1 | 10.255.0.21 | BR1-R | 10.255.0.41 |
| DIST-A2 | 10.255.0.22 | BR2-R | 10.255.0.42 |
| | | BR2-L3SW | 10.255.0.43 |

## 6. 고정 IP

| 장비 | IP | 게이트웨이 | 비고 |
|---|---|---|---|
| SRV-DHCP | 10.3.10.10 | 10.3.10.1 | 모든 본사 VLAN 풀 + LAP용 WLC 주소 |
| SRV-DNS | 10.3.10.11 | 10.3.10.1 | 내부 DNS (corp.local) |
| SRV-AAA | 10.3.20.10 | 10.3.20.1 | RADIUS(무선) + TACACS+(장비 로그인) |
| SRV-LOG-NTP | 10.3.20.11 | 10.3.20.1 | Syslog + NTP |
| NOC-ADMIN | 10.3.99.100 | 10.3.99.1 | 모든 장비 SSH 허용되는 유일한 대역 |
| WLC-B (관리) | 10.2.50.10 | 10.2.50.1 | |
| PRINTER-A | 10.1.30.50 | 10.1.30.1 | |
| BR2-PRN1 | 10.12.20.50 | 10.12.20.1 | |
| HQ-FW dmz | 203.0.113.193/27 | - | DMZ 게이트웨이 |
| DMZ-WEB | 203.0.113.200 | 203.0.113.193 | |
| DMZ-MAIL | 203.0.113.201 | 203.0.113.193 | |
| DMZ-DNS | 203.0.113.202 | 203.0.113.193 | 외부용 DNS |
| PUBLIC-DNS | 8.8.8.8 | 8.8.8.1 | |
| PUBLIC-WEB | 8.8.8.80 | 8.8.8.1 | |
| REMOTE-PC | 8.8.8.100 | 8.8.8.1 | DNS = 8.8.8.8 |
| 액세스 스위치 관리 IP | 각 MGMT VLAN의 .11 ~ | | ACC-A1=10.1.99.11, ACC-B1=10.2.99.11 … |
| 나머지 유선/무선 단말 | DHCP | | |

---

## 7. 포트 연결표 (78개)

케이블 기준: 같은 계층(스위치↔스위치, 라우터↔라우터, 라우터↔PC/서버 직결) = **Cross-Over** / 다른 계층(라우터·방화벽·WLC↔스위치, 스위치↔단말) = **Straight** / Serial = **DCE**를 HQ-EDGE2 쪽에서 먼저 클릭

### 인터넷 / WAN

| # | A | 포트 | B | 포트 | 케이블 |
|---|---|---|---|---|---|
| 1 | ISP-A | Gi0/0 | HQ-EDGE1 | Gi0/0 | Cross |
| 2 | ISP-A | Gi0/2 | BR1-R | Gi0/0/0 | Cross |
| 3 | ISP-A | Gi0/1 | ISP-B | Gi0/2 | Cross |
| 4 | ISP-B | Gi0/0 | HQ-EDGE2 | Gi0/0 | Cross |
| 5 | ISP-B | Gi0/1 | INET-CORE | Gi0/0 | Cross |
| 6 | INET-CORE | Gi0/2 | INET-SW | Gi0/1 | Straight |
| 7 | INET-SW | Fa0/1 | PUBLIC-DNS | Fa0 | Straight |
| 8 | INET-SW | Fa0/2 | PUBLIC-WEB | Fa0 | Straight |
| 9 | INET-SW | Fa0/3 | REMOTE-PC | Fa0 | Straight |
| 10 | HQ-EDGE2 | Se0/3/0 | BR2-R | Se0/3/0 | Serial DCE (HQ-EDGE2 쪽이 DCE) |

### 본사 코어

| # | A | 포트 | B | 포트 | 케이블 |
|---|---|---|---|---|---|
| 11 | HQ-EDGE1 | Gi0/1 | HQ-CORE1 | Gi1/0/1 | Straight |
| 12 | HQ-EDGE1 | Gi0/2 | HQ-CORE2 | Gi1/0/1 | Straight |
| 13 | HQ-EDGE2 | Gi0/1 | HQ-CORE1 | Gi1/0/2 | Straight |
| 14 | HQ-EDGE2 | Gi0/2 | HQ-CORE2 | Gi1/0/2 | Straight |
| 15 | HQ-CORE1 | Gi1/0/3 | HQ-CORE2 | Gi1/0/3 | Cross (Po1) |
| 16 | HQ-CORE1 | Gi1/0/4 | HQ-CORE2 | Gi1/0/4 | Cross (Po1) |
| 17 | HQ-CORE1 | Gi1/0/5 | DIST-A1 | Gi0/1 | Cross |
| 18 | HQ-CORE1 | Gi1/0/6 | DIST-A2 | Gi0/1 | Cross |
| 19 | HQ-CORE1 | Gi1/0/7 | DIST-B1 | Gi0/1 | Cross |
| 20 | HQ-CORE1 | Gi1/0/8 | DIST-B2 | Gi0/1 | Cross |
| 21 | HQ-CORE1 | Gi1/0/9 | DC-SW1 | Gi0/1 | Cross |
| 22 | HQ-CORE1 | Gi1/0/10 | DC-SW2 | Gi0/1 | Cross |
| 23 | HQ-CORE2 | Gi1/0/5 | DIST-A1 | Gi0/2 | Cross |
| 24 | HQ-CORE2 | Gi1/0/6 | DIST-A2 | Gi0/2 | Cross |
| 25 | HQ-CORE2 | Gi1/0/7 | DIST-B1 | Gi0/2 | Cross |
| 26 | HQ-CORE2 | Gi1/0/8 | DIST-B2 | Gi0/2 | Cross |
| 27 | HQ-CORE2 | Gi1/0/9 | DC-SW1 | Gi0/2 | Cross |
| 28 | HQ-CORE2 | Gi1/0/10 | DC-SW2 | Gi0/2 | Cross |
| 29 | HQ-CORE1 | Gi1/0/11 | HQ-FW | Gi1/1 | Straight |

### DMZ

| # | A | 포트 | B | 포트 | 케이블 |
|---|---|---|---|---|---|
| 30 | HQ-FW | Gi1/2 | DMZ-SW | Gi0/1 | Straight |
| 31 | DMZ-SW | Fa0/1 | DMZ-WEB | Fa0 | Straight |
| 32 | DMZ-SW | Fa0/2 | DMZ-MAIL | Fa0 | Straight |
| 33 | DMZ-SW | Fa0/3 | DMZ-DNS | Fa0 | Straight |

### 건물 A

| # | A | 포트 | B | 포트 | 케이블 |
|---|---|---|---|---|---|
| 34 | DIST-A1 | Fa0/23 | DIST-A2 | Fa0/23 | Cross (Po2) |
| 35 | DIST-A1 | Fa0/24 | DIST-A2 | Fa0/24 | Cross (Po2) |
| 36 | DIST-A1 | Fa0/1 | ACC-A1 | Gi0/1 | Cross |
| 37 | DIST-A1 | Fa0/2 | ACC-A2 | Gi0/1 | Cross |
| 38 | DIST-A1 | Fa0/3 | ACC-A3 | Gi0/1 | Cross |
| 39 | DIST-A2 | Fa0/1 | ACC-A1 | Gi0/2 | Cross |
| 40 | DIST-A2 | Fa0/2 | ACC-A2 | Gi0/2 | Cross |
| 41 | DIST-A2 | Fa0/3 | ACC-A3 | Gi0/2 | Cross |
| 42 | ACC-A1 | Fa0/1 | PC-HR1 | Fa0 | Straight |
| 43 | ACC-A1 | Fa0/11 | PC-FIN1 | Fa0 | Straight |
| 44 | ACC-A2 | Fa0/1 | PC-HR2 | Fa0 | Straight |
| 45 | ACC-A2 | Fa0/11 | PC-SALES1 | Fa0 | Straight |
| 46 | ACC-A3 | Fa0/1 | PC-FIN2 | Fa0 | Straight |
| 47 | ACC-A3 | Fa0/11 | PC-SALES2 | Fa0 | Straight |
| 48 | ACC-A3 | Fa0/20 | PRINTER-A | Fa0 | Straight |

### 건물 B

| # | A | 포트 | B | 포트 | 케이블 |
|---|---|---|---|---|---|
| 49 | DIST-B1 | Fa0/23 | DIST-B2 | Fa0/23 | Cross (Po2) |
| 50 | DIST-B1 | Fa0/24 | DIST-B2 | Fa0/24 | Cross (Po2) |
| 51 | DIST-B1 | Fa0/1 | ACC-B1 | Gi0/1 | Cross |
| 52 | DIST-B1 | Fa0/2 | ACC-B2 | Gi0/1 | Cross |
| 53 | DIST-B1 | Fa0/3 | ACC-B3 | Gi0/1 | Cross |
| 54 | DIST-B2 | Fa0/1 | ACC-B1 | Gi0/2 | Cross |
| 55 | DIST-B2 | Fa0/2 | ACC-B2 | Gi0/2 | Cross |
| 56 | DIST-B2 | Fa0/3 | ACC-B3 | Gi0/2 | Cross |
| 57 | ACC-B1 | Fa0/1 | PC-ENG1 | Fa0 | Straight |
| 58 | ACC-B1 | Fa0/11 | PC-DEV1 | Fa0 | Straight |
| 59 | ACC-B2 | Fa0/1 | PC-ENG2 | Fa0 | Straight |
| 60 | ACC-B2 | Fa0/11 | PC-DEV2 | Fa0 | Straight |
| 61 | ACC-B3 | Fa0/1 | LAP-1 | Gi0 | Straight |
| 62 | ACC-B3 | Fa0/2 | LAP-2 | Gi0 | Straight |
| 63 | ACC-B3 | Fa0/24 | WLC-B | Gi1 | Straight |
| - | LAPTOP-CORP1, LAPTOP-GUEST1, PHONE-1 | 무선 | | | 케이블 없음 |

### 데이터센터

| # | A | 포트 | B | 포트 | 케이블 |
|---|---|---|---|---|---|
| 64 | DC-SW1 | Fa0/23 | DC-SW2 | Fa0/23 | Cross (Po2) |
| 65 | DC-SW1 | Fa0/24 | DC-SW2 | Fa0/24 | Cross (Po2) |
| 66 | DC-SW1 | Fa0/1 | SRV-DHCP | Fa0 | Straight |
| 67 | DC-SW1 | Fa0/2 | SRV-DNS | Fa0 | Straight |
| 68 | DC-SW2 | Fa0/1 | SRV-AAA | Fa0 | Straight |
| 69 | DC-SW2 | Fa0/2 | SRV-LOG-NTP | Fa0 | Straight |
| 70 | DC-SW2 | Fa0/20 | NOC-ADMIN | Fa0 | Straight |

### 지사

| # | A | 포트 | B | 포트 | 케이블 |
|---|---|---|---|---|---|
| 71 | BR1-R | Gi0/0/1 | BR1-SW | Gi0/1 | Straight |
| 72 | BR1-SW | Fa0/1 | BR1-PC1 | Fa0 | Straight |
| 73 | BR1-SW | Fa0/2 | BR1-PC2 | Fa0 | Straight |
| 74 | BR2-R | Gi0/0 | BR2-L3SW | Gi0/1 | Straight |
| 75 | BR2-L3SW | Fa0/1 | BR2-ACC1 | Gi0/1 | Cross |
| 76 | BR2-ACC1 | Fa0/1 | BR2-PC1 | Fa0 | Straight |
| 77 | BR2-ACC1 | Fa0/2 | BR2-PC2 | Fa0 | Straight |
| 78 | BR2-ACC1 | Fa0/11 | BR2-PRN1 | Fa0 | Straight |

### 액세스 포트 VLAN 할당

| 스위치 | Fa0/1-10 | Fa0/11-19 | Fa0/20 | 그 외 |
|---|---|---|---|---|
| ACC-A1 | 110 HR | 120 FIN | 120 | 666 shutdown |
| ACC-A2 | 110 HR | 130 SALES | 130 | 666 |
| ACC-A3 | 120 FIN | 130 SALES | 130 (프린터) | 666 |
| ACC-B1 / ACC-B2 | 210 ENG | 220 DEV | 220 | 666 |
| ACC-B3 | 250 AP-MGMT (LAP) | - | - | Fa0/24 = WLC 트렁크 (230,240,250) |
| DC-SW1 | 310 | - | - | |
| DC-SW2 | 320 | - | 399 (NOC) | |
| BR1-SW | 10 | - | - | |
| BR2-ACC1 | 10 | 20 (프린터) | - | |

---

## 7-1. 모듈 장착 목록 (아직 장착 안 함 — 직접 장착)

장비 클릭 → Physical 탭 → **전원 스위치 OFF** → 모듈을 빈 슬롯으로 드래그 → 전원 ON

| 장비 | 모듈 | 이유 |
|---|---|---|
| HQ-EDGE2 | HWIC-2T ✅ 장착됨 (슬롯 3) | Serial 0/3/0 (지사2 링크, 케이블 #10) |
| BR2-R | HWIC-2T ✅ 장착됨 (슬롯 3) | Serial 0/3/0 (지사2 링크, 케이블 #10) |
| HQ-CORE1, HQ-CORE2 | AC-POWER-SUPPLY | 3650은 전원 모듈이 없으면 꺼진 상태 |
| LAP-1, LAP-2 | 전원 어댑터 (Physical 탭) | 2960은 PoE 미지원 |
| LAPTOP-CORP1, LAPTOP-GUEST1 | WPC300N (기존 유선 NIC 제거 후) | 무선 접속 |

## 7-2. 현재 Packet Tracer 상태 (2026-10-15)

- 저장 파일: `Network-Project.pkt` (이 폴더)
- 장비 60대 배치, 이름(Display Name) 변경 완료 — hostname 등 장비 설정은 전혀 안 함
- 케이블 78개 전부 연결 완료 (#10 Serial 포함)
- 화면 옵션: "Always Show Port Labels" 켜둠 (Options → Preferences)

## 8. 구현 요구사항 (무엇을 할지만, 방법은 직접)

### L2
- [ ] 모든 트렁크 Native VLAN 999, 필요한 VLAN만 허용
- [ ] Rapid-PVST+, STP Root = HSRP Active와 일치 (4장 표)
- [ ] 분배 스위치 쌍 / DC 쌍 = L2 LACP EtherChannel (Po2)
- [ ] 코어 쌍 = **L3** EtherChannel (Po1)
- [ ] 액세스 포트: Port-Security, PortFast, BPDU Guard
- [ ] DHCP Snooping + Dynamic ARP Inspection
- [ ] 안 쓰는 포트 VLAN 666 + shutdown

### L3 / 라우팅
- [ ] 분배·DC 스위치에서 HSRP (VLAN별로 Active 분산)
- [ ] OSPF 멀티 에어리어: Area 0 = 코어/엣지/코어-분배 링크, Area 10 = 건물 A, Area 20 = 건물 B, Area 30 = DC, Area 11 = 지사1 LAN
- [ ] ABR에서 요약: 10.1.0.0/16, 10.2.0.0/16, 10.3.0.0/16
- [ ] Area 10은 Totally Stubby
- [ ] 지사2 = EIGRP AS 100, HQ-EDGE2에서 OSPF ↔ EIGRP 상호 재분배
- [ ] 지사2 인터넷은 본사 경유 (센트럴 브레이크아웃)
- [ ] 엣지 → 내부로 기본 경로 전파

### WAN / 인터넷
- [ ] eBGP: HQ AS 65001 ↔ ISP-A AS 100, ISP-B AS 200 / ISP-A ↔ ISP-B, ISP-B ↔ INET AS 300
- [ ] iBGP: HQ-EDGE1 ↔ HQ-EDGE2 (루프백 기반)
- [ ] 203.0.113.0/24는 양쪽 다 광고, /26는 각자 자기 것만 광고 (회선 장애 시 자동 우회)
- [ ] 각 엣지에서 NAT/PAT (DMZ 대역은 NAT 제외)
- [ ] HQ-EDGE1 ↔ BR1-R: GRE 터널 + IPsec 암호화, 터널 위로 OSPF

### 보안
- [ ] ASA: inside / dmz 존, 외부 → DMZ-WEB(80), DMZ-MAIL(25,110), DMZ-DNS(53)만 허용, DMZ → 내부 차단
- [ ] 게스트 무선(240): 인터넷만, 내부 10.0.0.0/8 전부 차단
- [ ] HR(110) ↔ FIN(120) 서로 접근 불가, 둘 다 서버(310)는 허용
- [ ] 모든 장비 SSH v2만, VTY는 NOC-MGMT(10.3.99.0/24)에서만
- [ ] 장비 로그인 TACACS+ (SRV-AAA), 서버 다운 시 로컬 계정으로 폴백

### 서비스
- [ ] SRV-DHCP 하나로 본사 전체 VLAN 배포 (릴레이), LAP는 DHCP로 WLC 주소 받기
- [ ] 무선: SSID `CORP-WIFI` = WPA2-Enterprise (RADIUS), `GUEST-WIFI` = WPA2-PSK
- [ ] DNS: 내부 `www.corp.local` → DMZ-WEB, 외부 PUBLIC-DNS에서 `www.corp.com` → 203.0.113.200
- [ ] 모든 장비 NTP 동기화 + Syslog 전송

## 9. 최종 검증 시나리오

| # | 시나리오 | 기대 |
|---|---|---|
| 1 | PC-HR1 → PUBLIC-WEB 웹 접속 | 성공 (ISP-A 경유) |
| 2 | HQ-EDGE1 Gi0/0 shutdown 후 1번 반복 | ISP-B로 자동 우회 성공 |
| 3 | REMOTE-PC → `www.corp.com` | DMZ-WEB 페이지 |
| 4 | REMOTE-PC → 10.x 아무거나 | 실패 |
| 5 | BR1-PC1 ↔ BR2-PC1 ping | 성공 (터널 → 본사 → 시리얼) |
| 6 | PC-HR1 → PC-FIN1 ping | **실패** (ACL) |
| 7 | LAPTOP-GUEST1 → SRV-DNS(10.3.10.11) | **실패** / 인터넷은 성공 |
| 8 | LAPTOP-CORP1 접속 | RADIUS 인증 후 10.2.30.x 받음 |
| 9 | DIST-A1 전원 OFF | 건물 A 사용자 잠깐 끊긴 후 복구 (HSRP) |
| 10 | HQ-CORE1 전원 OFF | 전체 서비스 유지 |
| 11 | NOC-ADMIN → 아무 장비 SSH | 성공 / 다른 PC에서는 거부 |
| 12 | SRV-LOG-NTP Syslog 탭 | 모든 장비 로그가 쌓임 |

## 10. 확장 과제

1. IPv6 듀얼스택 + OSPFv3
2. 지사1 백업 회선 (ISP-B로 두 번째 터널)
3. ASA Active/Standby 이중화 (HQ-FW2 추가, CORE2 연결)
4. VoIP: 음성 VLAN + IP Phone + CME (건물 A)
5. NetFlow / SNMP 모니터링 서버 추가
