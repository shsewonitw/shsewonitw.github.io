---
layout: post
title: "[Daily morning study] Service Mesh와 Istio 개념"
description: >
  #daily morning study
category: 
    - dms
    - dms-devops
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## Service Mesh가 왜 필요한가

마이크로서비스 아키텍처에서는 수십~수백 개의 서비스가 서로 HTTP/gRPC로 통신한다. 이때 다음과 같은 공통 문제가 반복된다.

- **트래픽 제어**: 재시도, 타임아웃, 서킷 브레이커
- **보안**: 서비스 간 mTLS 인증/암호화
- **관찰 가능성**: 분산 트레이싱, 메트릭, 로그
- **로드밸런싱**: 고급 라우팅(가중치, 헤더 기반)

이런 기능을 각 서비스 코드에 직접 구현하면 언어마다 중복 구현이 생기고 유지보수가 어려워진다. Service Mesh는 이 로직을 **인프라 계층**으로 분리해 서비스 코드와 무관하게 적용한다.

---

## Service Mesh의 구조

Service Mesh는 **데이터 플레인(Data Plane)**과 **컨트롤 플레인(Control Plane)**으로 나뉜다.

| 구분 | 역할 | 예시 |
|------|------|------|
| 데이터 플레인 | 실제 트래픽을 처리하는 사이드카 프록시 | Envoy |
| 컨트롤 플레인 | 프록시의 설정과 정책을 중앙에서 관리 | Istiod |

**사이드카 패턴**: 각 Pod에 Envoy 프록시 컨테이너를 자동으로 주입해 모든 인바운드/아웃바운드 트래픽이 프록시를 통하도록 한다.

```
[Service A] → [Envoy (sidecar)] ──network──▶ [Envoy (sidecar)] → [Service B]
                    ↑                                    ↑
                    └──────────── Istiod ────────────────┘
                              (설정/정책 배포)
```

---

## Istio 주요 구성 요소

### Istiod (컨트롤 플레인)

Istio 1.5 이후 단일 바이너리로 통합됐다. 내부적으로 세 가지 기능을 포함한다.

- **Pilot**: 서비스 디스커버리, 트래픽 관리 규칙을 Envoy에 배포
- **Citadel**: 인증서 발급 및 갱신 (mTLS용)
- **Galley**: 설정 유효성 검사 및 분배

### Envoy 프록시 (데이터 플레인)

C++로 작성된 고성능 L7 프록시. Istio의 데이터 플레인으로 사용되며 다음을 처리한다.

- 동적 서비스 디스커버리
- HTTP/2, gRPC, WebSocket 지원
- 서킷 브레이커, 재시도, 타임아웃
- 요청별 메트릭 수집

---

## Istio 핵심 리소스

### VirtualService

트래픽 라우팅 규칙을 정의한다. Kubernetes의 Service 위에서 더 정교한 라우팅을 추가한다.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - match:
        - headers:
            end-user:
              exact: jason
      route:
        - destination:
            host: reviews
            subset: v2
    - route:
        - destination:
            host: reviews
            subset: v1
```

위 설정은 `end-user: jason` 헤더가 있는 요청만 v2로 보내고 나머지는 v1으로 라우팅한다.

### DestinationRule

라우팅 대상(subset)의 속성을 정의한다. 로드밸런싱 정책, 서킷 브레이커, TLS 설정 등을 포함한다.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
```

`outlierDetection`이 서킷 브레이커 역할을 한다. 5xx 오류가 5회 연속 발생하면 해당 인스턴스를 30초 동안 로드밸런싱에서 제외한다.

### PeerAuthentication

서비스 간 mTLS 정책을 설정한다.

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT  # 네임스페이스 내 모든 서비스 간 mTLS 강제
```

- `STRICT`: mTLS만 허용
- `PERMISSIVE`: 평문 트래픽과 mTLS 모두 허용 (마이그레이션 시 사용)
- `DISABLE`: mTLS 비활성화

---

## Istio 트래픽 관리 기능

### 카나리 배포

가중치 기반 라우팅으로 점진적 배포를 구현한다.

```yaml
http:
  - route:
      - destination:
          host: frontend
          subset: stable
        weight: 90
      - destination:
          host: frontend
          subset: canary
        weight: 10
```

### 폴트 인젝션 (Fault Injection)

카오스 엔지니어링 목적으로 의도적으로 지연이나 오류를 주입해 장애 내성을 테스트한다.

```yaml
http:
  - fault:
      delay:
        percentage:
          value: 10.0
        fixedDelay: 5s
      abort:
        percentage:
          value: 5.0
        httpStatus: 500
    route:
      - destination:
          host: ratings
```

전체 요청의 10%에 5초 지연을 주고, 5%는 500 오류를 반환한다.

---

## 관찰 가능성 연동

Istio는 별도 코드 수정 없이 다음 도구와 연동된다.

| 도구 | 역할 |
|------|------|
| Prometheus | Envoy가 자동으로 메트릭 노출 |
| Grafana | Istio 공식 대시보드 제공 |
| Jaeger / Zipkin | 분산 트레이싱 (B3 헤더 전파) |
| Kiali | 서비스 간 의존성 시각화 그래프 |

트레이싱을 제대로 활용하려면 애플리케이션이 수신한 B3 헤더(`x-request-id`, `x-b3-traceid` 등)를 다음 서비스 호출 시 그대로 전달해야 한다.

---

## Istio vs Linkerd

Istio 외에 **Linkerd**도 대표적인 Service Mesh 구현체다.

| 항목 | Istio | Linkerd |
|------|-------|---------|
| 데이터 플레인 | Envoy (C++) | Linkerd2-proxy (Rust) |
| 리소스 사용량 | 비교적 무거움 | 경량 |
| 기능 범위 | 풍부 (VM, 멀티클러스터 등) | Kubernetes에 집중 |
| 설정 복잡도 | 높음 | 낮음 |
| mTLS 기본값 | PERMISSIVE | 자동 활성화 |

단순한 mTLS와 기본 트래픽 관찰이 목적이면 Linkerd가 간편하고, 복잡한 트래픽 제어나 VM 통합이 필요하면 Istio가 적합하다.

---

## 정리

- Service Mesh는 서비스 간 통신 문제(보안, 트래픽, 관찰)를 코드 밖 인프라 계층으로 분리한다.
- Istio는 Envoy 사이드카 + Istiod 컨트롤 플레인 구조로 동작한다.
- VirtualService로 라우팅 규칙을, DestinationRule로 대상 속성을 정의한다.
- mTLS, 서킷 브레이커, 폴트 인젝션, 카나리 배포를 YAML 설정만으로 구현할 수 있다.
- 운영 복잡도가 높아지므로 도입 전 팀의 쿠버네티스 숙련도와 운영 여건을 먼저 고려해야 한다.
