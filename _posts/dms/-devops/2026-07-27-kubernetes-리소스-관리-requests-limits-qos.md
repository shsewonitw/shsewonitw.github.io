---
layout: post
title: "[Daily morning study] Kubernetes 리소스 관리: Requests, Limits, QoS 클래스"
description: >
  #daily morning study
category: 
    - dms
    - dms-devops
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 왜 리소스 관리가 필요한가

Kubernetes 클러스터는 여러 파드가 노드의 CPU와 메모리를 공유한다. 아무런 제한 없이 파드를 배포하면 특정 파드가 노드 자원을 독점해 다른 파드에 영향을 준다. 이를 "노이지 네이버(Noisy Neighbor)" 문제라고 한다.

리소스 요청(Requests)과 제한(Limits)을 설정하면 다음이 가능해진다.

- 스케줄러가 파드를 배치할 노드를 정확히 결정할 수 있음
- 특정 파드가 노드 자원을 독점하지 못하도록 막음
- QoS(Quality of Service) 클래스를 통해 자원 경합 시 우선순위를 결정

---

## Requests와 Limits

### Requests (요청)

파드가 실행되기 위해 **보장받아야 하는** 최소 리소스량이다. 스케줄러는 이 값을 기준으로 파드를 배치할 노드를 선택한다. 노드의 할당 가능한 자원(`Allocatable`)이 파드의 요청량보다 적으면 그 노드에 스케줄링되지 않는다.

### Limits (제한)

파드가 **사용할 수 있는 최대** 리소스량이다. 파드가 이 값을 초과하면 다음과 같이 처리된다.

- **CPU**: 스로틀링(Throttling)이 발생. CPU 사용량이 제한치를 넘으면 강제로 속도가 줄어들지만 파드는 종료되지 않는다.
- **메모리**: OOM(Out Of Memory) Killer가 개입해 파드가 강제 종료(OOMKilled)된다.

### 설정 예시

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: my-app:latest
    resources:
      requests:
        memory: "256Mi"
        cpu: "250m"
      limits:
        memory: "512Mi"
        cpu: "500m"
```

CPU 단위인 `250m`은 0.25 코어(250 밀리코어)를 의미한다. 메모리 단위 `Mi`는 MiB(메비바이트)다.

---

## CPU와 메모리의 압축성(Compressible) 차이

| 특성 | CPU | 메모리 |
| --- | --- | --- |
| 유형 | 압축 가능(Compressible) | 압축 불가(Incompressible) |
| 제한 초과 시 | 스로틀링 | OOMKilled (파드 종료) |
| 제한 권장 여부 | 경우에 따라 생략 가능 | 반드시 설정 권장 |

CPU는 제한을 초과해도 파드가 죽지 않고 느려지기만 한다. 반면 메모리는 한번 초과되면 프로세스가 강제 종료되기 때문에 더 신중하게 설정해야 한다.

---

## QoS 클래스

Kubernetes는 Requests와 Limits 설정에 따라 각 파드에 세 가지 QoS 클래스 중 하나를 자동으로 부여한다. 노드에 자원이 부족해 파드를 퇴거(Eviction)시켜야 할 때 이 클래스 순서대로 먼저 제거된다.

### 1. BestEffort (최하위 우선순위)

Requests와 Limits를 **아무것도 설정하지 않은** 파드다. 자원을 얼마나 쓸지 선언하지 않았기 때문에 자원 경합 시 가장 먼저 퇴거된다.

```yaml
resources: {}  # 아무것도 설정하지 않음
```

### 2. Burstable (중간 우선순위)

Requests와 Limits 중 **하나만 설정**하거나, 둘 다 설정했지만 **값이 다른** 파드다. 일반적으로 가장 많이 사용하는 설정이다.

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

평상시에는 Requests만큼 보장받고, 여유가 있을 때는 Limits까지 버스트해서 사용할 수 있다.

### 3. Guaranteed (최상위 우선순위)

Requests와 Limits가 **모든 컨테이너에서 동일하게 설정**된 파드다. 가장 마지막에 퇴거되며, 안정성이 중요한 핵심 서비스에 적합하다.

```yaml
resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

---

## LimitRange와 ResourceQuota

### LimitRange

네임스페이스 내 개별 파드나 컨테이너에 기본값(Default)과 최대/최소값을 강제한다. Requests/Limits를 명시하지 않은 파드에 자동으로 기본값이 적용된다.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
  - default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "250m"
      memory: "256Mi"
    type: Container
```

### ResourceQuota

네임스페이스 전체에서 소비할 수 있는 **총 리소스 합계**를 제한한다. 팀이나 프로젝트별로 클러스터 자원 사용량을 통제할 때 사용한다.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    pods: "20"
```

---

## 실전 설정 가이드

**1. 메모리 Limits는 반드시 설정한다.**
메모리 누수나 예외 상황에서 OOMKilled 없이 노드 전체를 죽이는 최악의 상황을 막아야 한다.

**2. CPU Limits는 신중하게 결정한다.**
CPU Limits를 너무 낮게 잡으면 정상 트래픽에서도 스로틀링이 발생해 레이턴시가 높아진다. 트래픽 패턴을 파악하고 실제 사용량에 여유를 더해 설정한다.

**3. Requests는 실제 평균 사용량 기준으로 설정한다.**
Prometheus + Grafana로 실제 파드 사용량을 측정한 뒤 설정하는 게 이상적이다. VPA(Vertical Pod Autoscaler)를 `Recommendation` 모드로 사용하면 권고치를 자동으로 계산해 준다.

**4. Guaranteed 클래스는 DB, 메시지 브로커 같은 상태 유지 서비스에 적용한다.**
Burstable은 웹 애플리케이션처럼 가변 트래픽을 처리하는 서비스에 적합하다.

---

## 노드 자원 계산 방식

스케줄러가 보는 노드의 사용 가능한 자원은 실제 물리 자원과 다르다.

```
Allocatable = Node Capacity - kube-reserved - system-reserved - eviction-threshold
```

노드 OS와 kubelet 자체가 사용하는 자원(`kube-reserved`, `system-reserved`)과 퇴거 임계값을 제외한 나머지가 파드에 할당 가능한 자원이다. 이 때문에 8GiB 메모리 노드에 8GiB를 요청하는 파드는 스케줄링되지 않는다.

```bash
# 노드의 Allocatable 확인
kubectl describe node <node-name> | grep -A 5 "Allocatable"
```

---

## 정리

| 개념 | 역할 |
| --- | --- |
| Requests | 스케줄링 기준, 보장되는 최소 자원 |
| Limits | 사용 가능한 최대 자원 (초과 시 스로틀링 또는 OOMKilled) |
| BestEffort | Requests/Limits 없음, 가장 먼저 퇴거 |
| Burstable | Requests < Limits, 중간 우선순위 |
| Guaranteed | Requests == Limits, 가장 마지막에 퇴거 |
| LimitRange | 파드/컨테이너 단위 기본값과 범위 강제 |
| ResourceQuota | 네임스페이스 전체 자원 합계 제한 |

Kubernetes 리소스 관리는 클러스터 안정성과 직결된다. Requests/Limits를 적절히 설정하고, LimitRange로 기본값을 강제하며, ResourceQuota로 팀 단위 사용량을 통제하는 세 가지를 함께 운용하는 것이 핵심이다.
