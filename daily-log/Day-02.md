# 2일차 — ASA 방화벽 설정 (outside/dmz/inside)

## 오늘 한 것
ASA0의 인터페이스 3개(outside/dmz/inside)에 IP와 보안 레벨을 설정하고,
DMZ 서버 3대(Web/Mail/DNS)를 외부에서 접근 가능하게 ACL로 열어줬다.
또한 Router0, Router1-A/B, PC 3대, DMZ 서버 3대까지 IP 주소를 채워서
1일차에 만든 물리적 배선이 실제로 통신되는지 확인했다.

## 설정한 것
- ASA0 인터페이스 3개: outside(레벨 0), dmz(레벨 50), inside(레벨 100)
- 정적 라우팅: ASA0 ↔ Router0 ↔ Router1-A 구간
- ACL: 외부 → DMZ 웹서버(80/443), 메일서버(25/110), DNS서버(53), ICMP만 허용
- Router0, Router1-A/B, PC0~2, DMZ 서버 3대 IP 주소 설정

## IP 주소 계획

| 장비 | 인터페이스 | IP |
|---|---|---|
| ASA0 | outside | 203.0.113.2/30 |
| ASA0 | inside | 10.0.0.1/29 |
| ASA0 | dmz | 172.16.0.1/24 |
| Router1-A | Gig0/0 | 10.0.0.2/29 |
| Web/Mail/DNS-Server | - | 172.16.0.10~12/24 |

## 실행 결과

<img width="442" height="248" alt="스크린샷 2026-08-21 102900" src="https://github.com/user-attachments/assets/c9adf744-fb58-4557-b538-c71f37520594" />

PC0에서 DMZ의 Web-Server까지 정상적으로 핑이 통하는 것을 확인했다.
(경로: PC0 → Router0 → ASA0 outside → ASA0 dmz → Web-Server)

## 트러블슈팅
처음 핑을 쳤을 때 4번 중 3번이 타임아웃되고 마지막 1번만 성공했다.
찾아보니 이건 ARP(주소 해석 프로토콜) 때문에 처음 통신할 때 흔히 발생하는
정상적인 현상이었다. 다시 핑을 쳐보니 ARP 캐시가 저장되어 4번 다 성공했다.

## 다음에 할 것
3일차 — 부서별 VLAN(영업/인사/IT/재무) 구성 및 트렁크 설정
