# Day 3 — 인터넷 연결 (BGP 멀티호밍 + NAT)

## 오늘 한 것
| 단계 | 내용 | 대상 장비 |
|---|---|---|
| 3-1 | 공인 구간 IP, 인터넷 서버 고정 IP | ISP-A, ISP-B, INET-CORE, HQ-EDGE1/2, 서버 3대 |
| 3-2 | eBGP (AS 65001 / 100 / 200 / 300), 공인 대역 광고 | ISP-A, ISP-B, INET-CORE, HQ-EDGE1/2 |
| 3-3 | 정적 기본 경로 + OSPF로 내부 전파 | HQ-EDGE1/2 |
| 3-4 | NAT/PAT (사설 10.0.0.0/8 → 공인 203.0.113.x) | HQ-EDGE1/2 |

## 구성 요약
| AS | 장비 | 광고하는 네트워크 |
|---|---|---|
| 65001 (본사) | HQ-EDGE1 | 203.0.113.0/24, 203.0.113.0/26 |
| 65001 (본사) | HQ-EDGE2 | 203.0.113.0/24, 203.0.113.64/26 |
| 100 | ISP-A | 100.0.2.0/30 (지사1 회선) |
| 200 | ISP-B | (중계만) |
| 300 | INET-CORE | 8.8.8.0/24 |

| 엣지 | NAT 주소 | 기본 경로 |
|---|---|---|
| HQ-EDGE1 | 203.0.113.1 | 100.0.1.1 (ISP-A) |
| HQ-EDGE2 | 203.0.113.65 | 200.0.1.1 (ISP-B) |

## 설계 포인트
- **/24 + /26 동시 광고:** 양쪽 엣지가 /24를 광고하고 각자 자기 /26만 추가 광고
  → 응답은 더 구체적인 /26을 따라 NAT한 엣지로 정확히 복귀 (Longest Match)
  → 한 회선 장애 시 /26 광고가 사라지고 /24로 우회
- **Null0 정적 경로:** BGP `network`는 라우팅 테이블에 같은 경로가 있어야 광고되므로 광고용 경로 생성
- **default-information originate:** 엣지의 정적 기본 경로를 OSPF로 내부에 전파.
  기본 경로가 있을 때만 광고하므로 회선 장애 시 자동으로 광고 중단 → 다른 엣지로 전환
- **PAT (overload):** 공인 주소 하나를 포트 번호로 나눠 여러 PC가 공유

## 트러블슈팅 / PT 한계

### 1. `default-information originate metric 10`이 안 들어감
- 증상: `% Invalid input detected at '^' marker` (`metric` 위치)
- 확인: `router ospf 1`에서 `default-information originate ?` → `<cr>`만 나옴
- 원인: Packet Tracer OSPF는 metric 옵션 미지원 (실제 IOS는 지원)
- 해결: 옵션 없이 양쪽 엣지에 설정 → **Active-Active** 구조로 변경
- 영향: 주 회선/백업 구분 대신 두 회선 동시 사용. NAT와 /26 광고 덕분에 응답 경로 정상, 장애 우회도 정상

### 2. `ip route ... Null0` 입력 시 경고
- 메시지: `%Default route without gateway, if not a point-to-point interface, may impact performance`
- 원인: 포트 이름으로 지정한 정적 경로에 붙는 일반 경고. Null0은 ARP를 하지 않으므로 실제 영향 없음
- 확인: `show ip route static` → `S 203.0.113.0/24 is directly connected, Null0`

## 검증 결과
- `show ip bgp summary` (각 장비)

  <img width="848" height="781" alt="image" src="https://github.com/user-attachments/assets/a0dafd8e-74ea-4971-ac0d-a532bcce8484" />
  
- ISP-A `show ip route bgp`

  <img width="874" height="692" alt="스크린샷 2026-10-26 155332" src="https://github.com/user-attachments/assets/7957d94f-6522-4fa1-a277-f77b7b90f197" />

- HQ-CORE1 `show ip route`

  <img width="865" height="688" alt="스크린샷 2026-10-26 155358" src="https://github.com/user-attachments/assets/c653dd0f-e0fa-43fc-8d5c-3c90ed67719d" />
  
- PC-HR1 `ping 8.8.8.8`

  <img width="833" height="297" alt="image" src="https://github.com/user-attachments/assets/389098e8-aa6f-4346-85ff-3d1355a59dc2" />

- HQ-EDGE1/2 `show ip nat translations`

  <img width="915" height="731" alt="image" src="https://github.com/user-attachments/assets/9b96bb04-1bb0-4063-8a79-7f0e4f30c7c6" />

- 장애 테스트: HQ-EDGE1 Gi0/0 shutdown → PC-HR1 `ping 8.8.8.8`

  <img width="911" height="735" alt="image" src="https://github.com/user-attachments/assets/6fbc6f18-b48d-481e-8bf4-d08111f15560" />

## 남은 이슈 / 다음에 할 것
- 엣지 간 iBGP 미구성 (PT의 update-source / next-hop-self 지원 여부 확인 필요, 현재 구성에선 없어도 동작)
- Day 4: 지사1 GRE over IPsec + OSPF, 지사2 Serial + EIGRP + 재분배
