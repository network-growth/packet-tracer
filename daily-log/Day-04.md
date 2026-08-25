# 3일차 — 부서별 VLAN 구성 및 인터-VLAN 라우팅

## 오늘 한 것
영업/인사/IT/재무/서버팜을 각각 독립된 VLAN으로 분리하고, Core-SW-A를
L3 스위치로 동작시켜(ip routing) VLAN 간 라우팅이 되도록 구성했다.
Router1-A/B ↔ Core-SW-A/B 구간도 새로 IP를 배정해서 전체 경로를 연결했다.

## VLAN 구성

| VLAN | 이름 | 대역 |
|---|---|---|
| 10 | SALES | 192.168.10.0/24 |
| 20 | HR | 192.168.20.0/24 |
| 30 | IT | 192.168.30.0/24 |
| 40 | FINANCE | 192.168.40.0/24 |
| 99 | SERVERFARM | 192.168.99.0/24 |

## 설정한 것
- 부서 스위치 5대에 VLAN 생성 + 액세스포트/트렁크포트 지정
- Core-SW-A에 VLAN 5개 SVI(가상 인터페이스) 생성 및 `ip routing` 활성화
- Router1-A/B ↔ Core-SW-A/B 구간 라우팅 인터페이스(10.1.0.0/30, 10.1.1.0/30) 구성
- ASA0에 ICMP 상태 추적(`inspect icmp`) 추가

## 실행 결과
<img width="557" height="189" alt="스크린샷 2026-08-25 111412" src="https://github.com/user-attachments/assets/c3692365-f7f4-4f00-a44e-6948e8d9c4ee" />

Sales-PC1(VLAN10)에서 DMZ의 Web-Server까지 전체 경로(스위치→코어→라우터→ASA→DMZ)가
정상적으로 통신되는 것을 확인했다.

## 트러블슈팅
1. `Router1-A`에 부서 VLAN 대역으로 가는 라우트만 넣고, 기본 경로(default route)를
   빠뜨려서 DMZ 등 나머지 목적지로 가는 길을 몰라 `Destination host unreachable`
   에러가 발생했다. `ip route 0.0.0.0 0.0.0.0 10.0.0.1`로 해결.
2. 라우팅이 다 맞았는데도 핑이 `Request timed out`으로 계속 실패했다.
   원인은 Cisco ASA가 기본적으로 ICMP는 상태 추적(stateful inspection) 대상이
   아니라서, 응답 패킷이 돌아올 때 자동으로 막혔던 것이었다.
   `policy-map global_policy → inspect icmp`로 ICMP도 상태 추적하게 만들어 해결했다.

## 다음에 할 것
4일차 — 내부 경계(Router1-A/B)와 코어(Core-SW-A/B) 이중화(HSRP) 구성
