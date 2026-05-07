---
layout: post
title: "[Daily morning study] AWS Route 53과 DNS 라우팅 정책"
description: >
  #daily morning study
category: 
    - dms
    - dms-cloud
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## Route 53이란

Route 53은 AWS에서 제공하는 DNS(Domain Name System) 웹 서비스다. 단순한 DNS 역할을 넘어 헬스 체크, 트래픽 라우팅, 도메인 등록까지 지원한다.

이름의 유래: DNS 서비스가 TCP/UDP 포트 53을 사용하는 데서 따왔다.

**Route 53의 주요 기능:**
- **도메인 등록**: Route 53에서 직접 도메인 구매 및 관리 가능
- **DNS 확인(Resolution)**: 도메인 이름을 IP 주소로 변환
- **헬스 체크(Health Check)**: 엔드포인트 상태를 주기적으로 확인
- **트래픽 라우팅**: 다양한 라우팅 정책으로 트래픽 분산

---

## DNS 레코드 타입

| 레코드 타입 | 설명 |
|-----------|------|
| A | 도메인 → IPv4 주소 |
| AAAA | 도메인 → IPv6 주소 |
| CNAME | 도메인 → 다른 도메인 (루트 도메인에는 사용 불가) |
| Alias | AWS 리소스(ALB, CloudFront 등)와 매핑. 루트 도메인에도 사용 가능 |
| MX | 이메일 서버 지정 |
| TXT | 텍스트 정보 (SPF, 도메인 소유권 확인 등) |

**CNAME vs Alias 비교**

CNAME은 루트 도메인(`example.com`)에 사용할 수 없다. Alias는 AWS 전용이지만 루트 도메인에도 적용 가능하고, Route 53이 내부적으로 처리하기 때문에 쿼리 비용이 무료다.

---

## 라우팅 정책(Routing Policy) 7가지

### 1. Simple Routing (단순 라우팅)

가장 기본적인 정책. 하나의 도메인에 하나의 IP 또는 값을 반환한다.

```
example.com → 192.0.2.1
```

- 헬스 체크 불가
- 여러 값을 넣으면 클라이언트에게 랜덤으로 하나 반환

---

### 2. Weighted Routing (가중치 기반 라우팅)

각 리소스에 가중치(weight)를 지정해서 비율대로 트래픽을 분산한다.

```
서버 A (weight: 80) → 80% 트래픽
서버 B (weight: 20) → 20% 트래픽
```

- A/B 테스트, 블루/그린 배포에 활용
- 특정 레코드의 weight를 0으로 설정하면 해당 리소스로 트래픽을 보내지 않음
- 헬스 체크와 함께 사용 가능

---

### 3. Latency-based Routing (지연 시간 기반 라우팅)

사용자와 AWS 리전 간의 네트워크 지연 시간(latency)을 기준으로 가장 빠른 리전으로 라우팅한다.

```
한국 사용자 → ap-northeast-2 (서울)
미국 사용자 → us-east-1 (버지니아)
```

- 글로벌 서비스에서 사용자 경험 향상에 유용
- 실제 물리적 거리가 아닌 네트워크 레이턴시 기준

---

### 4. Failover Routing (장애 조치 라우팅)

Primary 엔드포인트에 장애가 발생하면 Secondary로 자동 전환한다.

```
Primary: 메인 서버 (헬스 체크 통과 시)
Secondary: 대기 서버 또는 S3 정적 페이지
```

- 반드시 헬스 체크 설정이 필요
- Active-Passive 고가용성 구성에 사용

---

### 5. Geolocation Routing (지리적 위치 기반 라우팅)

사용자의 지리적 위치(국가, 대륙)를 기준으로 라우팅한다.

```
대한민국 사용자 → 한국 서버
유럽 사용자 → 유럽 서버
기본값 → 글로벌 서버
```

- 지역별 콘텐츠 제공, 규정 준수(데이터 주권)에 활용
- Latency Routing과 혼동 주의: Geolocation은 위치 기준, Latency는 속도 기준

---

### 6. Geoproximity Routing (근접성 기반 라우팅)

지리적 위치 기반이지만 bias 값으로 특정 리전에 더 많은 트래픽을 유도할 수 있다.

```
bias: +50 → 해당 리전 커버리지 확장 (더 많은 트래픽 흡수)
bias: -50 → 해당 리전 커버리지 축소 (더 적은 트래픽)
```

- Traffic Flow 기능 사용 필요
- Geolocation보다 세밀한 트래픽 조정 가능

---

### 7. Multi-Value Answer Routing (다중 값 응답 라우팅)

여러 IP 주소를 반환하면서 각각에 헬스 체크를 수행한다.

```
example.com → [192.0.2.1, 192.0.2.2, 192.0.2.3] (헬스 체크를 통과한 것만)
```

- Simple Routing의 무작위 반환과 달리 헬스 체크 지원
- 클라이언트 사이드 로드 밸런싱 효과
- ELB를 대체하는 기능은 아님

---

## 라우팅 정책 비교 요약

| 정책 | 키워드 | 헬스 체크 | 주요 사용 사례 |
|------|--------|-----------|--------------|
| Simple | 단순 연결 | 불가 | 단일 리소스 |
| Weighted | 비율 분산 | 가능 | A/B 테스트, 블루/그린 |
| Latency | 속도 기준 | 가능 | 글로벌 서비스 최적화 |
| Failover | 장애 대응 | 필수 | Active-Passive HA |
| Geolocation | 국가/대륙 | 가능 | 지역별 콘텐츠 |
| Geoproximity | 위치 + 편향 | 가능 | 세밀한 트래픽 제어 |
| Multi-Value | 여러 IP | 가능 | 클라이언트 사이드 LB |

---

## 헬스 체크(Health Check)

Route 53은 엔드포인트의 상태를 주기적으로 확인하는 헬스 체크를 지원한다.

**헬스 체크 종류:**

- **Endpoint health check**: 특정 IP 또는 도메인의 HTTP/HTTPS/TCP 응답 확인
- **Calculated health check**: 여러 헬스 체크 결과를 조합 (AND/OR 조건)
- **CloudWatch alarm health check**: CloudWatch 알람 상태를 기준으로 판단

**헬스 체크 동작 방식:**

- 전 세계 여러 Route 53 헬스 체커가 동시에 엔드포인트에 요청
- 18% 이상의 체커가 정상 응답을 확인하면 healthy로 판정
- 비공개 VPC 내부 리소스는 외부에서 직접 체크 불가 → CloudWatch 알람 활용

---

## TTL (Time To Live)

DNS 응답이 캐시에 유지되는 시간이다.

| TTL | 특징 |
|-----|------|
| 높은 TTL (예: 24시간) | DNS 쿼리 비용 절감, 레코드 변경 반영이 느림 |
| 낮은 TTL (예: 60초) | 레코드 변경이 빠르게 반영, DNS 쿼리 비용 증가 |

레코드 변경을 계획할 때는 미리 TTL을 낮게 설정해두는 것이 좋다. 변경 완료 후 다시 높은 TTL로 되돌릴 수 있다.

---

## DNS 이름 확인 흐름

```
사용자
  → 로컬 DNS 캐시 확인
  → (없으면) ISP의 재귀 리졸버에 쿼리
  → Root Nameserver (.com TLD 확인)
  → TLD Nameserver (Route 53 Nameserver 주소 반환)
  → Route 53 Authoritative Nameserver
  → 최종 IP 반환
```

Route 53은 Authoritative DNS 역할을 담당한다.

---

## 실무 활용 패턴

**멀티 리전 Active-Active**

```
Latency Routing + Health Check 조합
→ 각 리전 서버에 레이턴시 기반 라우팅 설정
→ 특정 리전 장애 시 자동으로 다른 리전으로 전환
```

**블루/그린 배포**

```
Weighted Routing 활용
→ 초기:    Blue(100), Green(0)
→ 점진적: Blue(50),  Green(50)
→ 완료:   Blue(0),   Green(100)
```

**재해 복구(DR)**

```
Failover Routing 활용
→ Primary:   메인 서비스
→ Secondary: S3 정적 에러 페이지 또는 백업 서버
```
