# Enterprise HQ-Branch Network

> Cisco Packet Tracer로 설계한 본사-지사 대형 네트워크 — DMZ, 부서별 VLAN, 이중화(HSRP), WAN(OSPF)까지 포함

## 진행 단계
- [x] 1단계: 전체 토폴로지 설계 및 물리 배선 (34대)
- [x] 2단계: ASA 방화벽 설정 (outside/dmz/inside, 보안 레벨, ACL)
- [x] 3단계: 부서별 VLAN 및 트렁크 설정
- [ ] 4단계: 내부 경계/코어 이중화 설정 (HSRP)
- [ ] 5단계: 본사-지사 WAN 라우팅 (OSPF)
- [ ] 6단계: 전체 통합 테스트 및 장애 시뮬레이션
- [ ] 7단계: 최종 문서 정리

## 구조
```
[외부존] → [ASA0] → DMZ (Web/Mail/DNS)
              ↓
     Router1-A/B → Core-SW-A/B (이중화)
              ↓
   Sales/HR/IT/Finance/ServerFarm VLAN
              ↓
        Branch-RTR → Branch 사이트
```

## 파일
- `topology/enterprise-hq-branch-network.pkt` — Packet Tracer 원본
- `topology/topology-full.png` — 전체 구조 스크린샷
- `daily-log/` — 일자별 작업 기록
