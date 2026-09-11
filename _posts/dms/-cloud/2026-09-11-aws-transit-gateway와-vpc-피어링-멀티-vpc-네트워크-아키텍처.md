---
layout: post
title: "[Daily morning study] AWS Transit Gateway와 VPC 피어링 — 멀티 VPC 네트워크 아키텍처"
description: >
  #daily morning study
category: 
    - dms
    - dms-cloud
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 왜 멀티 VPC 구조가 필요한가

프로젝트 초기엔 VPC 하나로 모든 걸 해결한다. 그런데 조직이 커지면 VPC를 여러 개 운영하는 상황이 생긴다.

- 팀별 또는 환경별(dev/staging/prod)로 네트워크 격리
- 보안 요구 사항에 따라 서비스마다 별도 AWS 계정 분리
- M&A 후 다른 조직의 VPC와 연결 필요

이 때 VPC 간 통신을 어떻게 연결하느냐가 문제다.

---

## VPC 피어링 (VPC Peering)

VPC 피어링은 두 VPC 사이에 직접 네트워크 연결을 만드는 가장 기본적인 방법이다.

```
VPC-A (10.0.0.0/16) ←── 피어링 ──→ VPC-B (10.1.0.0/16)
```

### 동작 방식

1. 한쪽 VPC에서 피어링 연결 요청
2. 상대 VPC에서 수락
3. 양쪽 VPC의 라우팅 테이블에 상대방 CIDR 경로 추가
4. 보안 그룹에서 상대방 IP 범위 허용

같은 리전, 다른 리전(Inter-Region Peering), 다른 AWS 계정 간 연결도 가능하다.

### 핵심 제약: 비전이적(Non-Transitive)

VPC 피어링의 가장 큰 제약은 **비전이성**이다.

```
VPC-A ── 피어링 ── VPC-B ── 피어링 ── VPC-C

A → B 통신 가능
B → C 통신 가능
A → C 통신 불가능 (B를 경유해서 갈 수 없다)
```

A와 C가 통신하려면 A-C 간 피어링을 별도로 만들어야 한다.

### VPC가 5개라면?

모든 VPC 쌍을 연결하려면 필요한 피어링 수 = `n(n-1)/2`

| VPC 수 | 필요한 피어링 수 |
|--------|----------------|
| 3      | 3              |
| 5      | 10             |
| 10     | 45             |
| 20     | 190            |

VPC가 늘어날수록 관리가 기하급수적으로 복잡해진다. 라우팅 테이블도 각각 손으로 관리해야 한다.

---

## Transit Gateway

Transit Gateway는 이 문제를 해결하기 위해 2018년에 출시된 서비스다. 여러 VPC와 온프레미스 네트워크를 하나의 허브에 연결하는 **리전 레벨의 네트워크 라우터** 역할을 한다.

```
        ┌────────────────────────────┐
        │       Transit Gateway      │
        │   (중앙 라우팅 허브)        │
        └──┬────┬────┬────┬─────────┘
           │    │    │    │
         VPC-A VPC-B VPC-C VPN/Direct Connect
```

### 핵심 개념

**Attachment(연결)**
- VPC를 Transit Gateway에 연결하는 단위
- VPC Attachment, VPN Attachment, Direct Connect Gateway Attachment, Peering Attachment 종류가 있다

**Route Table(라우팅 테이블)**
- Transit Gateway 자체에 라우팅 테이블이 있다
- 어떤 Attachment에서 들어온 트래픽을 어디로 보낼지 정의

**Association & Propagation**
- Association: 특정 Attachment가 어떤 라우팅 테이블을 사용할지 연결
- Propagation: Attachment의 CIDR을 라우팅 테이블에 자동 등록

### 전이성 지원

Transit Gateway를 쓰면 비전이성 문제가 해결된다.

```
VPC-A ──┐
VPC-B ──┼── Transit Gateway ── (라우팅 테이블에 따라 모든 VPC가 통신 가능)
VPC-C ──┘
```

VPC 10개를 연결하려면 피어링 45개 대신 Attachment 10개만 만들면 된다.

---

## Transit Gateway 라우팅 구성 예시

### 기본 설정 (모든 VPC 간 통신 허용)

```
Transit Gateway Route Table:
  10.0.0.0/16 → VPC-A attachment
  10.1.0.0/16 → VPC-B attachment
  10.2.0.0/16 → VPC-C attachment
```

모든 VPC에 같은 라우팅 테이블을 연결하면 삼각형 구조 없이 통신 가능.

### 격리 구성 (dev/prod 분리)

보안 요구 사항에 따라 dev VPC와 prod VPC가 서로 통신하지 못하게 막고 싶을 때, 라우팅 테이블을 분리하면 된다.

```
prod-route-table:
  association: prod-vpc-a, prod-vpc-b
  propagation: prod-vpc-a, prod-vpc-b
  → prod끼리만 통신 가능

dev-route-table:
  association: dev-vpc-a, dev-vpc-b
  propagation: dev-vpc-a, dev-vpc-b
  → dev끼리만 통신 가능
```

### 공유 서비스 구성

DNS, 모니터링, 보안 도구 같은 공유 서비스 VPC를 만들고 모든 VPC에서 접근 가능하게 하되, 일반 VPC끼리는 서로 접근 못하게 하는 패턴이다.

```
shared-services VPC ──→ Transit Gateway
prod-vpc-a        ──→ Transit Gateway
prod-vpc-b        ──→ Transit Gateway

shared-rt:
  prod-vpc-a, prod-vpc-b propagation 포함
  → shared 서비스가 모든 VPC에 접근 가능

prod-rt:
  shared-services만 propagation 포함
  → 일반 VPC는 shared 서비스만 접근 가능, 서로는 격리
```

---

## VPN & Direct Connect 연결

Transit Gateway는 온프레미스 네트워크와의 연결도 중앙화할 수 있다.

**Site-to-Site VPN**

```
온프레미스 데이터센터 ──── VPN (IPSec) ──── Transit Gateway
                                              ├── VPC-A
                                              ├── VPC-B
                                              └── VPC-C
```

온프레미스 - Transit Gateway VPN 하나로 모든 VPC에 접근 가능. 이전에는 VPC마다 VPN을 따로 설정해야 했다.

**Direct Connect**

Direct Connect Gateway를 Transit Gateway에 연결하면 전용선으로 연결된 온프레미스가 여러 VPC에 접근할 수 있다.

---

## 리전 간 연결 (Inter-Region Peering)

Transit Gateway끼리 피어링 연결이 가능하다.

```
리전 A                          리전 B
┌──────────────────────┐        ┌──────────────────────┐
│  TGW-A               │        │  TGW-B               │
│  VPC-A1, VPC-A2 연결 │◄──────►│  VPC-B1, VPC-B2 연결 │
└──────────────────────┘        └──────────────────────┘
```

리전 A의 모든 VPC와 리전 B의 모든 VPC가 단일 TGW Peering으로 통신 가능. 대신 이 구간은 인터넷을 경유하지 않고 AWS 백본 네트워크를 통해 암호화 전송된다.

---

## VPC 피어링 vs Transit Gateway 비교

| 항목 | VPC 피어링 | Transit Gateway |
|------|-----------|----------------|
| 연결 방식 | 1:1 직접 연결 | 허브-앤-스포크 |
| 전이성 | 비전이적 (없음) | 지원 |
| 관리 복잡도 | VPC 증가 시 급격히 증가 | 중앙 집중 관리 |
| 대역폭 제한 | 없음 (VPC 기본 한도) | 최대 50 Gbps/연결 |
| 비용 | 리전 간 데이터 전송 비용 | Attachment 비용 + 데이터 처리 비용 |
| 지연 시간 | 더 낮음 (직접 연결) | 약간 더 높음 |
| 온프레미스 연결 | 불가 | VPN/Direct Connect 통합 가능 |
| 멀티 계정 | 가능 | 가능 (RAM으로 공유) |

### 선택 기준

```
VPC 수가 적고 (3개 이하), 단순 연결이면
    → VPC 피어링 (비용 저렴, 설정 간단)

VPC 수가 많거나 온프레미스 연결이 필요하면
    → Transit Gateway

트래픽 격리 정책이 복잡하면
    → Transit Gateway (라우팅 테이블 분리)
```

---

## Resource Access Manager (RAM) — 계정 간 공유

Transit Gateway는 기본적으로 같은 계정의 VPC만 연결할 수 있다. 다른 AWS 계정의 VPC를 연결하려면 **AWS Resource Access Manager(RAM)**를 사용한다.

1. TGW를 생성한 계정(마스터)에서 RAM으로 TGW를 다른 계정에 공유
2. 공유받은 계정에서 해당 TGW에 자신의 VPC를 Attachment로 연결
3. 마스터 계정의 라우팅 테이블에 경로 추가

AWS Organizations를 사용하면 조직 전체에 자동으로 공유할 수 있어 대규모 멀티 계정 환경에서 편리하다.

---

## 실무 아키텍처 패턴

### 일반적인 엔터프라이즈 구성

```
온프레미스 (Direct Connect)
         ↓
Transit Gateway
    ├── 공유 서비스 VPC (DNS, 모니터링, 보안)
    ├── Ingress VPC (외부 트래픽 진입점, ALB)
    ├── Prod VPC-1 (서비스 A)
    ├── Prod VPC-2 (서비스 B)
    └── Dev VPC (개발 환경)
```

- Ingress VPC는 외부 인터넷 트래픽을 받아 Transit Gateway를 통해 Prod VPC로 전달
- Prod VPC는 퍼블릭 서브넷 없이 프라이빗 서브넷만 운영 (최소 인터넷 노출)
- Dev VPC는 Prod VPC와 라우팅 테이블로 격리

이 패턴을 **중앙화된 인그레스(Centralized Ingress)** 아키텍처라고 부른다.
