---
layout: post
title: "[Daily morning study] Kubernetes HPA (Horizontal Pod Autoscaler)"
description: >
  #daily morning study
category: 
    - dms
    - dms-devops
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## HPA란?

HPA(Horizontal Pod Autoscaler)는 Kubernetes에서 Deployment나 StatefulSet의 파드 수를 자동으로 조정하는 컨트롤러다. CPU 사용률, 메모리 사용률, 또는 커스텀 메트릭에 따라 파드를 늘리거나 줄인다.

수직 확장(Vertical Scaling)이 파드에 더 많은 리소스(CPU·메모리)를 주는 방식이라면, 수평 확장(Horizontal Scaling)은 파드 개수 자체를 늘리는 방식이다. HPA는 후자를 자동화한다.

## 동작 원리

HPA는 컨트롤 루프(Control Loop)로 동작한다. 기본적으로 15초마다 Metrics Server로부터 현재 메트릭을 가져와 목표값과 비교하고, 필요한 파드 수를 계산한다.

파드 수 계산 공식:

```
desiredReplicas = ceil[currentReplicas × (currentMetricValue / desiredMetricValue)]
```

예: 현재 파드 2개, 목표 CPU 50%, 현재 평균 CPU 80%

```
desiredReplicas = ceil[2 × (80 / 50)] = ceil[3.2] = 4
```

파드를 4개로 늘린다.

## 구성 요소

| 구성 요소 | 역할 |
|-----------|------|
| Metrics Server | 파드의 CPU/메모리 메트릭 수집 |
| HPA Controller | 메트릭 기반으로 파드 수 결정 |
| Deployment | 실제 파드 수를 조정하는 대상 |

## HPA 설정 방법

**기본 YAML (CPU 기반)**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

- `minReplicas`: 최소 파드 수
- `maxReplicas`: 최대 파드 수
- `averageUtilization`: 목표 CPU 사용률(%)

**kubectl 명령어로 빠르게 생성**

```bash
kubectl autoscale deployment my-app --cpu-percent=50 --min=2 --max=10
```

**상태 확인**

```bash
kubectl get hpa
kubectl describe hpa my-app-hpa
```

## 메트릭 종류 (v2 API)

**Resource 메트릭** — CPU, 메모리 같은 리소스 기반

```yaml
metrics:
- type: Resource
  resource:
    name: memory
    target:
      type: AverageValue
      averageValue: 500Mi
```

**External 메트릭** — 클러스터 외부 메트릭 (SQS 큐 길이 등)

```yaml
metrics:
- type: External
  external:
    metric:
      name: sqs_queue_length
    target:
      type: Value
      value: "100"
```

**Pods 메트릭** — 파드 자체의 커스텀 메트릭

```yaml
metrics:
- type: Pods
  pods:
    metric:
      name: http_requests_per_second
    target:
      type: AverageValue
      averageValue: "100"
```

## 스케일링 동작 제어 (behavior)

급격한 스케일 업/다운을 방지하기 위해 `behavior` 필드로 속도를 조절할 수 있다.

```yaml
spec:
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300   # 5분 안정화 구간
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60               # 1분에 최대 10%씩 줄임
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Pods
        value: 4
        periodSeconds: 15               # 15초마다 최대 4개씩 늘림
```

스케일 다운은 갑자기 줄이면 트래픽 처리에 문제가 생길 수 있어서 안정화 구간을 길게 잡는다. 스케일 업은 빠르게 반응해야 하므로 짧게 설정하는 게 일반적이다.

## 전제 조건

**1. Metrics Server 설치**

HPA가 동작하려면 Metrics Server가 클러스터에 설치되어 있어야 한다. 설치가 안 되어 있으면 `kubectl get hpa`에서 `<unknown>/50%`처럼 메트릭을 못 읽는 상태가 된다.

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

kubectl top pods
kubectl top nodes
```

**2. resources.requests 설정**

Deployment의 파드 스펙에 `resources.requests`가 명시되어야 CPU 기반 HPA가 정상 동작한다. requests가 없으면 CPU 사용률 계산 기준이 없어서 HPA가 작동하지 않는다.

```yaml
resources:
  requests:
    cpu: "200m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

## HPA vs VPA

| 항목 | HPA | VPA (Vertical Pod Autoscaler) |
|------|-----|-------------------------------|
| 확장 방식 | 파드 수 증가 | 파드 리소스(CPU/메모리) 증가 |
| 재시작 여부 | 불필요 | 보통 파드 재시작 필요 |
| 적합한 상황 | 무상태(stateless) 앱 | 단일 인스턴스 또는 상태 있는 앱 |
| 사용 빈도 | 높음 | 낮음 |

HPA와 VPA를 같이 쓸 수 있지만, CPU/메모리 메트릭 기반으로 둘 다 활성화하면 충돌이 생긴다. 보통 HPA는 커스텀 메트릭, VPA는 리소스 조정 용도로 분리해서 사용한다.

## 실무에서 주의할 점

- **스케일 다운 지연**: 기본적으로 스케일 다운은 5분 안정화 구간을 가진다. 트래픽이 빠져도 바로 줄이지 않는다.
- **Cold Start 문제**: 파드가 새로 뜨는 데 시간이 걸리는 앱이라면 `minReplicas`를 넉넉히 잡거나 Readiness Probe를 철저히 설정해야 한다.
- **Metrics Server 신뢰도**: 메트릭 수집이 일시적으로 실패하면 HPA는 스케일링을 멈춘다. 알람 설정을 권장한다.
- **minReplicas: 0 불가**: 기본 HPA로는 0으로 줄일 수 없다. 완전 종료가 필요하면 KEDA(Kubernetes Event-Driven Autoscaling)를 써야 한다.
