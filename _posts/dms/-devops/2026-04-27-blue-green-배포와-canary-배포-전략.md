---
layout: post
title: "[Daily morning study] Blue-Green 배포와 Canary 배포 전략"
description: >
  #daily morning study
category: 
    - dms
    - dms-devops
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---
# Blue-Green 배포와 Canary 배포 전략

프로덕션 환경에 새 버전을 배포할 때 핵심 목표는 두 가지다: **다운타임 없이** 배포하고, 문제가 생기면 **빠르게 롤백**하는 것. Blue-Green과 Canary는 이 목표를 달성하는 대표적인 배포 전략이다.

---

## 1. Blue-Green 배포

### 개념

두 개의 동일한 프로덕션 환경(Blue, Green)을 유지한다. 한쪽이 현재 라이브 환경이고, 다른 쪽은 새 버전을 배포하는 스테이징 역할을 한다.

```
[현재 상태]
사용자 → 로드밸런서 → Blue (v1, 라이브)
                    → Green (v1, 유휴)

[배포 중]
사용자 → 로드밸런서 → Blue (v1, 라이브)
                    → Green (v2, 배포 완료, 테스트 중)

[스위칭 후]
사용자 → 로드밸런서 → Blue (v1, 유휴 / 롤백 대기)
                    → Green (v2, 라이브)
```

### 배포 절차

1. Green 환경에 v2 배포 및 smoke test 실행
2. 로드밸런서의 트래픽을 Blue → Green으로 전환 (원자적 스위칭)
3. 일정 시간 모니터링
4. 문제 없으면 Blue 환경을 다음 배포를 위한 스탠바이로 전환
5. 문제 발생 시 로드밸런서를 Blue로 즉시 되돌림 (롤백)

### 장점

- 전환이 순간적으로 이루어지므로 다운타임이 사실상 0에 가깝다
- 롤백이 트래픽 스위칭 한 번으로 끝나서 매우 빠르다
- 두 환경이 완전히 분리되어 있어 테스트가 깔끔하다

### 단점

- 동일한 인프라를 두 벌 유지해야 하므로 비용이 두 배
- DB 스키마 변경이 동반되는 경우 양쪽 환경이 같은 DB를 공유하면 호환성 문제가 생길 수 있다
- 트래픽이 100%씩 전환되기 때문에 새 버전의 문제가 전체 사용자에게 즉시 영향을 준다

### DB 마이그레이션과의 연계

Blue-Green에서 DB 변경이 필요할 때는 **Expand-Contract 패턴**을 쓴다.

| 단계 | 설명 |
|------|------|
| Expand | 새 컬럼/테이블 추가. v1도 v2도 모두 동작 가능한 상태 유지 |
| 스위칭 | Green(v2)으로 트래픽 전환 |
| Contract | 더 이상 사용하지 않는 구 컬럼/테이블 제거 |

---

## 2. Canary 배포

### 개념

광부들이 유독가스를 탐지하기 위해 카나리아 새를 먼저 갱도에 보낸 것에서 유래했다. 새 버전을 전체 사용자가 아닌 **일부 소수 트래픽에만 먼저 노출**시켜 안정성을 검증한 뒤 점진적으로 확대한다.

```
[배포 초기]
사용자 100% → v1 (95%)
             → v2 (5%)

[검증 후 확대]
사용자 100% → v1 (50%)
             → v2 (50%)

[배포 완료]
사용자 100% → v2 (100%)
```

### 배포 절차

1. v2를 소수 인스턴스(예: 전체의 5~10%)에 배포
2. 로드밸런서에서 일부 트래픽만 v2로 라우팅
3. 에러율, 레이턴시, 비즈니스 지표 모니터링
4. 지표가 정상이면 트래픽 비율을 점진적으로 올림 (5% → 20% → 50% → 100%)
5. 문제 발생 시 v2 인스턴스를 제거하고 v1으로 완전히 복귀

### 트래픽 분산 방식

| 방식 | 설명 | 예시 도구 |
|------|------|-----------|
| 가중치 기반 | 전체 요청 중 일정 비율을 v2로 전송 | Nginx, AWS ALB |
| 사용자 기반 | 특정 사용자 그룹(내부 직원, 베타 유저)만 v2 | Feature Flag |
| 지역 기반 | 특정 리전 사용자에게만 v2 노출 | Cloudflare, AWS Route 53 |

### Nginx 가중치 설정 예시

```nginx
upstream backend {
    server v1.example.com weight=95;
    server v2.example.com weight=5;
}

server {
    location / {
        proxy_pass http://backend;
    }
}
```

### Kubernetes에서 Canary 배포

Kubernetes에서는 레이블 셀렉터와 레플리카 수를 활용해 간단하게 구현할 수 있다.

```yaml
# v1 Deployment: 9개 레플리카
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v1
spec:
  replicas: 9
  selector:
    matchLabels:
      app: myapp
      version: v1
  template:
    metadata:
      labels:
        app: myapp
        version: v1
    spec:
      containers:
      - name: app
        image: myapp:v1
---
# v2 Deployment: 1개 레플리카 (10% 트래픽)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      version: v2
  template:
    metadata:
      labels:
        app: myapp
        version: v2
    spec:
      containers:
      - name: app
        image: myapp:v2
---
# Service는 version 구분 없이 app 레이블만 셀렉트
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
spec:
  selector:
    app: myapp  # v1, v2 모두 포함
```

Argo Rollouts나 Flagger 같은 도구를 쓰면 자동으로 지표를 모니터링하며 트래픽 비율을 조절해준다.

### 장점

- 소수 사용자에게만 먼저 노출되므로 장애 영향 범위가 제한된다
- 실제 프로덕션 트래픽으로 충분한 검증 후 전체 배포 가능
- A/B 테스트와 결합하면 비즈니스 지표도 함께 검증할 수 있다

### 단점

- Blue-Green에 비해 롤백이 즉각적이지 않고 절차가 필요하다
- 두 버전이 동시에 운영되는 기간 동안 API 하위 호환성 유지가 필수
- 모니터링 및 트래픽 분산 인프라 설계가 더 복잡하다

---

## 3. 두 전략 비교

| 항목 | Blue-Green | Canary |
|------|-----------|--------|
| 전환 방식 | 전체 트래픽 한 번에 스위칭 | 점진적으로 트래픽 비율 증가 |
| 롤백 속도 | 즉시 (트래픽 스위칭) | 비교적 느림 |
| 장애 영향 범위 | 전환 시 전체 사용자 | 초기에는 소수 사용자만 |
| 인프라 비용 | 두 배 (2세트 유지) | 점진적 확장으로 비교적 낮음 |
| 적합한 경우 | 빠른 릴리즈, 인프라 비용 감당 가능 시 | 대규모 서비스, 안전한 점진적 검증 필요 시 |

---

## 4. Rolling Update와의 차이

배포 전략 중 가장 기본인 **Rolling Update**도 자주 비교된다.

```
[Rolling Update]
인스턴스: [v1] [v1] [v1] [v1]
           ↓
          [v2] [v1] [v1] [v1]  # 하나씩 교체
           ↓
          [v2] [v2] [v1] [v1]
           ↓
          [v2] [v2] [v2] [v2]  # 완료
```

Rolling Update는 인스턴스를 하나씩 새 버전으로 교체하는 방식이다. Canary와 달리 트래픽 비율을 세밀하게 제어하기 어렵고, 롤백 시 전체 인스턴스를 다시 v1으로 교체해야 하므로 느리다. 반면 추가 인프라 없이 구현할 수 있어 가장 널리 쓰이는 기본 전략이다.

---

## 5. 배포 전략 선택 기준

- **다운타임 0 + 빠른 롤백 우선** → Blue-Green
- **안전한 점진적 검증 우선** → Canary
- **비용 절감 + 단순함 우선** → Rolling Update
- **특정 사용자 그룹 실험** → Feature Flag + Canary 조합

실제 대형 서비스에서는 이 전략들을 조합해서 쓴다. 예를 들어 내부 테스터에게 Feature Flag로 먼저 노출한 뒤, Canary로 5%까지 확대하고, 안정성 확인 후 Blue-Green 스위칭으로 전체 배포하는 식이다.
