---
layout: post
title: "[Daily morning study] Kubernetes Operator 패턴 — Custom Resource와 Controller"
description: >
  #daily morning study
category: 
    - dms
    - dms-devops
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## Kubernetes Operator 패턴

### Operator란?

Kubernetes는 기본적으로 Deployment, StatefulSet, Service 같은 내장 리소스를 제공하지만, 복잡한 상태를 가진 애플리케이션(데이터베이스 클러스터, 메시지 큐, ML 파이프라인 등)을 운영하는 데 필요한 도메인 지식까지 내장하고 있지 않다.

**Operator 패턴**은 이 문제를 해결하기 위해 등장했다.

> Operator = CRD (Custom Resource Definition) + Custom Controller

특정 애플리케이션의 운영 지식을 코드로 구현해서, 사람이 수동으로 해야 했던 작업(롤링 업그레이드, 백업, 장애 복구 등)을 자동화하는 Kubernetes 확장 방식이다.

---

### 왜 Operator가 필요한가?

간단한 웹 서버는 Deployment만으로 충분하다. 그런데 Elasticsearch나 PostgreSQL 클러스터를 운영한다고 생각해보면:

- 노드 추가 → 샤드 재배분
- 버전 업그레이드 → 순서 있는 롤링 업데이트
- 장애 발생 → 특정 노드 제외 후 페일오버
- 주기적 스냅샷 백업

이런 절차는 단순한 `replicas: 3` 설정 수준이 아니다. Operator는 이 운영 절차를 컨트롤러 코드에 녹여서 K8s가 스스로 처리하게 만든다.

---

### CRD — API 확장

CRD(Custom Resource Definition)는 K8s API에 새로운 리소스 타입을 추가한다. 적용하면 `kubectl get myapps` 같은 명령이 동작한다.

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: myapps.example.com
spec:
  group: example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                replicas:
                  type: integer
                  minimum: 1
                image:
                  type: string
  scope: Namespaced
  names:
    plural: myapps
    singular: myapp
    kind: MyApp
```

CRD 등록 후, 이 타입으로 **Custom Resource(CR)**를 만들 수 있다:

```yaml
apiVersion: example.com/v1
kind: MyApp
metadata:
  name: my-app-instance
  namespace: default
spec:
  replicas: 3
  image: my-image:v1.2.3
```

이 YAML은 etcd에 저장되고, 컨트롤러가 읽어서 실제 Deployment, Service 등을 생성한다.

---

### Custom Controller — Reconciliation Loop

컨트롤러의 핵심은 **Reconciliation Loop**다. 관찰 → 차이 계산 → 행동을 반복하며 원하는 상태(Desired State)를 유지한다.

```
Observe → Diff → Act → Observe → ...
```

Go 기반 예시 (controller-runtime 라이브러리):

```go
func (r *MyAppReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 1. CR 조회
    var myApp myv1.MyApp
    if err := r.Get(ctx, req.NamespacedName, &myApp); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 2. 현재 Deployment 확인
    var deployment appsv1.Deployment
    err := r.Get(ctx, req.NamespacedName, &deployment)
    if errors.IsNotFound(err) {
        // 3. 없으면 생성
        newDeploy := r.buildDeployment(&myApp)
        return ctrl.Result{}, r.Create(ctx, newDeploy)
    }

    // 4. 있으면 원하는 상태와 비교 후 업데이트
    if *deployment.Spec.Replicas != myApp.Spec.Replicas {
        deployment.Spec.Replicas = &myApp.Spec.Replicas
        return ctrl.Result{}, r.Update(ctx, &deployment)
    }

    return ctrl.Result{}, nil
}
```

**Reconcile은 반드시 멱등성(Idempotent)을 보장해야 한다.** 동일한 상태에서 여러 번 실행해도 결과가 같아야 한다.

---

### Operator 개발 도구

#### Operator SDK

Red Hat이 만든 도구. Go / Helm / Ansible 기반 Operator를 지원한다.

```bash
# Go 기반 초기화
operator-sdk init --domain example.com --repo github.com/example/my-operator
operator-sdk create api --group apps --version v1 --kind MyApp --resource --controller
```

#### Kubebuilder

K8s SIG에서 관리하는 공식 스캐폴딩 도구.

```bash
kubebuilder init --domain example.com
kubebuilder create api --group apps --version v1 --kind MyApp
```

두 도구 모두 내부적으로 **controller-runtime** 라이브러리를 사용하며 구조가 비슷하다.

---

### Spec vs Status

| 필드 | 역할 | 누가 쓰나 |
|------|------|-----------|
| `spec` | 원하는 상태 (Desired State) | 사용자 |
| `status` | 관찰된 현재 상태 | 컨트롤러 |

컨트롤러는 작업 결과를 `status`에 기록한다:

```go
myApp.Status.Phase = "Running"
myApp.Status.ReadyReplicas = 3
r.Status().Update(ctx, &myApp)
```

```yaml
status:
  phase: Running
  readyReplicas: 3
  conditions:
    - type: Ready
      status: "True"
      lastTransitionTime: "2026-08-14T09:00:00Z"
```

---

### Finalizer — 삭제 흐름 제어

리소스 삭제 전에 외부 리소스 정리(예: 클라우드 볼륨 삭제, 외부 DB 해제)가 필요할 때 **Finalizer**를 사용한다.

```go
const myFinalizer = "example.com/finalizer"

// CR 생성 시 Finalizer 추가
controllerutil.AddFinalizer(&myApp, myFinalizer)
r.Update(ctx, &myApp)

// 삭제 요청 감지
if !myApp.DeletionTimestamp.IsZero() {
    // 정리 작업 수행
    if err := r.cleanupExternalResources(&myApp); err != nil {
        return ctrl.Result{}, err
    }
    // Finalizer 제거 → K8s가 실제 삭제 진행
    controllerutil.RemoveFinalizer(&myApp, myFinalizer)
    r.Update(ctx, &myApp)
}
```

Finalizer가 남아 있으면 K8s는 오브젝트를 실제로 삭제하지 않는다. `DeletionTimestamp`만 찍어두고 기다린다. 컨트롤러가 정리 작업을 마치고 Finalizer를 제거해야 비로소 etcd에서 삭제된다.

---

### Owner Reference — 가비지 컬렉션

컨트롤러가 생성한 하위 리소스(Deployment, Service 등)에 **OwnerReference**를 달면, CR이 삭제될 때 하위 리소스도 자동으로 삭제된다.

```go
ctrl.SetControllerReference(&myApp, deployment, r.Scheme)
```

이렇게 하면 직접 Finalizer 로직 없이도 종속 리소스가 깔끔하게 정리된다.

---

### Operator Maturity Model

CoreOS(현 Red Hat)가 정의한 Operator 성숙도 모델:

| Level | 이름 | 지원 기능 |
|-------|------|-----------|
| Level 1 | Basic Install | 자동 설치 및 설정 |
| Level 2 | Seamless Upgrades | 패치 및 마이너 버전 업그레이드 |
| Level 3 | Full Lifecycle | 백업, 복구, 장애 대응 |
| Level 4 | Deep Insights | 메트릭, 알림, 로그 분석 통합 |
| Level 5 | Auto Pilot | 수평 확장, 이상 감지, 자동 조정 |

단순한 Helm Chart는 Level 1에 해당한다. 완전한 Operator는 Level 3 이상을 목표로 하며, 복잡한 상태 관리와 복구 절차를 코드로 구현한다.

---

### 실제로 쓰이는 Operator 예시

| Operator | 역할 |
|----------|------|
| Prometheus Operator | `PrometheusRule`, `ServiceMonitor` CRD로 모니터링 설정 자동화 |
| Cert-Manager | `Certificate` CRD로 TLS 인증서 자동 발급 및 갱신 |
| ArgoCD | `Application` CRD로 GitOps 기반 배포 관리 |
| Strimzi | Kafka 클러스터 생성·업그레이드·토픽 관리 자동화 |
| CloudNativePG | PostgreSQL 클러스터 HA 구성 및 자동 페일오버 |

---

### 정리

- Operator는 **CRD + Custom Controller** 조합으로 K8s를 도메인별로 확장하는 패턴이다.
- 컨트롤러는 **Reconciliation Loop**를 통해 원하는 상태와 현재 상태의 차이를 지속적으로 맞춰나간다.
- `spec`은 사용자가 원하는 상태, `status`는 컨트롤러가 기록하는 현재 상태다.
- Finalizer로 삭제 전 정리 작업을 제어하고, OwnerReference로 하위 리소스의 가비지 컬렉션을 자동화한다.
- 개발 도구는 **Operator SDK**나 **Kubebuilder** 중 하나를 선택하면 된다.
