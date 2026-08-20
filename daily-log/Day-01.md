# 1일차 — 대형 엔터프라이즈 네트워크 토폴로지 설계 및 배선

## 오늘 한 것
지난 Zero-Trust 프로젝트가 소규모였어서, 이번엔 실제 회사 네트워크에 가까운 규모로
본사(HQ)-지사(Branch) 2개 사이트 구조를 처음부터 설계하고 물리적으로 다 배선했다.
설정(IP, VLAN, 라우팅 등)은 아직 없고, 순수하게 토폴로지 구조만 완성한 단계다.

## 전체 구조
```
[외부존] → [ASA0] → DMZ (Web/Mail/DNS)
              ↓
        [Switch-ASA-In] → Router1-A/B (이중화)
              ↓
        Core-SW-A / Core-SW-B (트렁크로 이중화)
              ↓
    Sales / HR / IT / Finance / ServerFarm (각 코어에 이중 업링크)
              ↓
        Branch-RTR → Branch-SW → Branch PC x2 (WAN 지사)
```

## 배치한 장비 (총 34대)
- 외부존: PC 3, Switch 1, Router 1
- 방화벽/DMZ: ASA 1, Switch 1, Server 3
- 내부 경계: Switch 1, Router 2 (이중화)
- 코어: Switch 2 (이중화)
- 부서 Access: Switch 5, PC 8, Server 2 (서버팜)
- 지사: Router 1, Switch 1, PC 2

## 설계 의도
- **ASA 보안 레벨**: outside(0) < dmz(50) < inside(100)로, 낮은 곳에서 높은 곳으로는
  기본 차단되게 설계. 웹서버가 뚫려도 내부망까지 못 번지게 하는 게 목적.
- **이중화**: 내부 경계 라우터, 코어 스위치, 부서 업링크를 전부 이중으로 구성해서
  어느 한 지점이 죽어도 통신이 유지되도록 설계했다.
- **지사 연결**: 본사와 별도 사이트를 WAN으로 연결해서, 나중에 OSPF로 전체를
  하나의 라우팅 도메인으로 묶을 계획이다.

## 다음에 할 것
2단계 — ASA 인터페이스 설정(outside/dmz/inside) 및 라우팅
