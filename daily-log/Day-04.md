# 4일차 — 내부 경계/코어 이중화 (HSRP)

## 오늘 한 것
Router1-A/B, Core-SW-A/B 양쪽에 HSRP(Hot Standby Router Protocol)를 적용해서,
한쪽 경로가 죽어도 반대편이 자동으로 게이트웨이 역할을 이어받도록 구성했다.

## 설정한 것
- 부서 VLAN(10/20/30/40/99) 각각에 가상 게이트웨이 IP 부여
  - Core-SW-A: 실제 IP `.2`, 우선순위 110(Active)
  - Core-SW-B: 실제 IP `.3`, 우선순위 90(Standby)
  - PC들의 게이트웨이는 가상 IP `.1` 그대로 유지
- Router1-A/B ↔ Switch-ASA-In 구간도 동일하게 가상 IP(10.0.0.4) 구성
- ASA0의 내부망 라우트를 실제 IP(10.0.0.2)에서 가상 IP(10.0.0.4)로 변경

## 실행 결과
<img width="652" height="129" alt="스크린샷 2026-08-25 152258" src="https://github.com/user-attachments/assets/b9df743d-e997-4730-b7bb-abdb200d97a3" />

<img width="650" height="129" alt="스크린샷 2026-08-25 152534" src="https://github.com/user-attachments/assets/ca052a92-58ee-4a73-ae43-38ec946febc7" />

<img width="650" height="78" alt="스크린샷 2026-08-25 152541" src="https://github.com/user-attachments/assets/80aed91f-401c-4ad9-a6ae-11d478ba9f4f" />

<img width="650" height="85" alt="스크린샷 2026-08-25 152546" src="https://github.com/user-attachments/assets/4618907b-24ea-46f3-901c-688fc206bb3f" />

<img width="510" height="284" alt="스크린샷 2026-08-25 152401" src="https://github.com/user-attachments/assets/d7c93548-a8da-437f-9916-e79cbeda40e8" />

<img width="494" height="140" alt="스크린샷 2026-08-25 152434" src="https://github.com/user-attachments/assets/5fc62bcf-af4a-4bc3-b61d-7860767ee76b" />

Core-SW-A가 Active, Core-SW-B가 Standby로 정상 동작하는 것을 확인했다.

## 트러블슈팅
- `standby X track 인터페이스이름 감소값` 명령어(업링크 장애 시 우선순위 자동 하향)를
  시도했으나 Packet Tracer의 3560 시뮬레이션 IOS가 이 기능(및 `track` 오브젝트 방식)을
  지원하지 않아 `Unrecognized command` 에러가 반복 발생했다.
  실제 Cisco 장비와 Packet Tracer 사이의 명령어 지원 범위 차이를 직접 확인한 케이스였다.
- 이 한계로 인해, 이번 구성은 "업링크 장애 시 자동 양보"까지는 구현하지 못했고,
  스위치/라우터 자체 다운에 대한 이중화까지만 검증했다.

## 프로젝트 한계 (솔직하게 기록)
Packet Tracer 환경의 명령어 지원 제한으로 `track` 기반 업링크 감시는 제외했다.
실제 장비(또는 GNS3 등 실제 IOS 에뮬레이션)라면 이 부분까지 완전하게 구현 가능하다.

## 다음에 할 것
5일차 — 본사-지사 WAN 연결 및 OSPF 라우팅
