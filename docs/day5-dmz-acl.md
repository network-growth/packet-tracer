# Day 5 — ASA DMZ + 내부 ACL

## 오늘 한 것
| 단계 | 내용 | 대상 장비 |
|---|---|---|
| 5-1 | ASA 존 구성 (inside 100 / dmz 50), 기본 경로, 코어에 DMZ 경로 + OSPF 재분배 | HQ-FW, HQ-CORE1 |
| 5-2 | DMZ 서버 3대 (웹/메일/DNS), DNS 레코드 (www.corp.com, www.corp.local) | DMZ 서버, PUBLIC-DNS, SRV-DNS |
| 5-3 | ASA 정책: DMZ로는 웹·메일·DNS만, ping은 내부에서만 | HQ-FW |
| 5-4 | HR(VLAN 110) ↔ FIN(VLAN 120) 상호 차단 | DIST-A1, DIST-A2 |
| 5-5 | 게스트 무선(VLAN 240) 내부망 차단, 인터넷만 허용 | DIST-B1, DIST-B2 |

## 구성 요약
| ASA 포트 | nameif | 보안 레벨 | IP |
|---|---|---|---|
| Gi1/1 | inside | 100 | 10.0.0.70/30 (→ HQ-CORE1 10.0.0.69) |
| Gi1/2 | dmz | 50 | 203.0.113.193/27 |

| DMZ 서버 | IP | 허용 서비스 |
|---|---|---|
| DMZ-WEB | 203.0.113.200 | HTTP (80) |
| DMZ-MAIL | 203.0.113.201 | SMTP (25), POP3 (110) |
| DMZ-DNS | 203.0.113.202 | DNS (53/UDP) |

| ACL | 장비 / 적용 | 내용 |
|---|---|---|
| 110 | DIST-A1/A2, VLAN 110 in | HR → FIN 차단 |
| 120 | DIST-A1/A2, VLAN 120 in | FIN → HR 차단 |
| 140 | DIST-B1/B2, VLAN 240 in | DHCP·내부 DNS만 허용, 10.0.0.0/8 차단, 인터넷 허용 |

## 설계 포인트
- **DMZ 공인 IP 직접 사용:** DMZ 서버는 203.0.113.192/27 공인 주소라 NAT 불필요
- **Longest Match로 Null0 우회:** 엣지의 Null0 /24보다 코어가 OSPF로 광고한 /27이 더 구체적이라 DMZ 트래픽이 ASA로 전달됨
- **ASA 보안 레벨:** 높은 → 낮은 레벨 기본 허용, 낮은 → 높은 레벨 기본 차단 → DMZ에서 내부로 먼저 연결 불가
- **ASA 상태 기반 검사:** TCP/UDP 응답은 자동 허용, ICMP 응답은 추적하지 않아 echo-reply를 별도 허용
- **양쪽 분배 스위치에 같은 ACL:** HSRP로 게이트웨이가 넘어가도 차단 유지

## 트러블슈팅

### 1. `ip access-list extended` 명령이 안 들어감
- 증상: 분배 스위치(3560)에서 이름 ACL 생성 명령 미동작
- 해결: 번호 ACL(100~199 확장)로 변경 — 동작은 동일
- 정리: 이름 ACL `ip access-group HR-IN in` ↔ 번호 ACL `ip access-group 110 in`

### 2. ASA ACL 마스크 형식
- 주의: IOS는 와일드카드(`0.255.255.255`), ASA는 일반 서브넷 마스크(`255.0.0.0`)

## 검증 결과
- HQ-FW `show interface ip brief` / `show nameif` → (inside 100, dmz 50)
- HQ-EDGE1 `show ip route` → (O E2 203.0.113.192/27)
- REMOTE-PC 브라우저 `http://www.corp.com` → (DMZ-WEB 페이지)
- REMOTE-PC `ping 203.0.113.200` → (실패 = 정상)
- PC-HR1 브라우저 `http://www.corp.local` → (DMZ-WEB 페이지)
- PC-HR1 → PC-FIN1 ping → (실패 = 정상), PC-HR1 → 10.3.10.10 → (성공)
- DIST-A1 `show access-lists` → (110, 120 매칭 수)

## 남은 이슈 / 다음에 할 것
- ASA 단일 장비 → 장애 시 DMZ 서비스 중단 (확장 과제: Active/Standby)
- 게스트 격리(ACL 140) 실제 확인은 6일차 무선 연결 후
- Day 6: WLC 무선, RADIUS, TACACS+, SSH 제한, NTP/Syslog
