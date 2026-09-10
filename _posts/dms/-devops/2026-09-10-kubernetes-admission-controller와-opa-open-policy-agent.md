---
layout: post
title: "[Daily morning study] Kubernetes Admission Controller와 OPA(Open Policy Agent)"
description: >
  #daily morning study
category: 
    - dms
    - dms-devops
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## Admission Controller란

Kubernetes API 서버로 들어오는 요청은 인증(Authentication) → 인가(Authorization) → Admission 순서로 처리된다.

Admission Controller는 이 세 번째 단계에서 동작하며, 오브젝트가 클러스터에 실제로 반영되기 **전에** 요청을 가로채서 검증하거나 변경하는 플러그인이다.

```
kubectl apply → API Server → Authn → Authz → [Admission] → etcd 저장
```

두 가지 타입이 있다.

| 타입 | 역할 |
|------|------|
| Mutating Admission | 요청을 수정 (예: 기본값 주입, 사이드카 자동 추가) |
| Validating Admission | 요청을 검증 후 허용 또는 거부 |

Mutating이 먼저 실행된 후 Validating이 실행된다.

---

## 빌트인 Admission Controller

Kubernetes에는 기본으로 내장된 Admission Controller들이 있다.

- **NamespaceLifecycle**: 삭제 중인 네임스페이스에 리소스 생성 금지
- **LimitRanger**: 리소스 요청/제한 기본값 적용
- **ServiceAccount**: 파드에 서비스 계정 자동 마운트
- **ResourceQuota**: 네임스페이스별 리소스 총량 제한
- **PodSecurity**: Pod Security Standards 적용 (PSP 대체)
- **MutatingAdmissionWebhook**: 외부 웹훅 호출 (변경)
- **ValidatingAdmissionWebhook**: 외부 웹훅 호출 (검증)

---

## Webhook 기반 Admission Controller

빌트인으로는 복잡한 정책 표현에 한계가 있다. 이를 해결하기 위해 Kubernetes는 외부 HTTP 서버로 요청을 위임하는 **Webhook** 방식을 지원한다.

### 동작 흐름

```
kubectl apply
    ↓
API Server
    ↓
MutatingAdmissionWebhook 설정 확인
    ↓
외부 웹훅 서버로 AdmissionReview 전송 (HTTP POST)
    ↓
웹훅 서버: 허용(allow) / 거부(deny) / 패치(patch) 응답
    ↓
ValidatingAdmissionWebhook 설정 확인
    ↓
최종 결정
```

### MutatingWebhookConfiguration 예시

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: my-mutating-webhook
webhooks:
  - name: inject-sidecar.example.com
    clientConfig:
      service:
        name: webhook-service
        namespace: webhook-system
        path: "/mutate"
    rules:
      - operations: ["CREATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
    admissionReviewVersions: ["v1"]
    sideEffects: None
    failurePolicy: Fail   # 웹훅 실패 시 요청 거부
```

---

## OPA(Open Policy Agent)란

OPA는 정책(Policy)을 코드로 표현하고 집행하는 범용 정책 엔진이다. 쿠버네티스에 종속되지 않으며 서비스 메시, API 게이트웨이, 마이크로서비스 등 다양한 환경에서 활용된다.

### 핵심 구성요소

| 구성요소 | 설명 |
|----------|------|
| Rego | OPA 전용 정책 언어 |
| Policy | Rego로 작성된 규칙 집합 |
| Input | 평가 대상 데이터 (JSON) |
| Data | 정책 판단에 참고하는 외부 데이터 |

### Rego 언어 기초

Rego는 선언형 쿼리 언어다. `deny` 규칙이 하나라도 참이면 요청이 거부된다.

```rego
package kubernetes.admission

# 특권 컨테이너 금지
deny[msg] {
    input.request.kind.kind == "Pod"
    container := input.request.object.spec.containers[_]
    container.securityContext.privileged == true
    msg := sprintf("컨테이너 '%v'에 privileged 모드가 허용되지 않습니다", [container.name])
}

# latest 태그 이미지 금지
deny[msg] {
    input.request.kind.kind == "Pod"
    container := input.request.object.spec.containers[_]
    endswith(container.image, ":latest")
    msg := sprintf("컨테이너 '%v'에 ':latest' 태그 이미지는 사용할 수 없습니다", [container.name])
}
```

---

## Gatekeeper: OPA의 Kubernetes 통합

**Gatekeeper**는 OPA를 Kubernetes Admission Controller로 통합하는 프로젝트다. CRD(Custom Resource Definition) 기반으로 정책을 선언적으로 관리한다.

### 주요 CRD

| CRD | 역할 |
|-----|------|
| ConstraintTemplate | Rego 정책 정의 및 CRD 생성 |
| Constraint | 실제 정책 인스턴스 (대상 리소스 지정) |

### ConstraintTemplate 예시

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels

        violation[{"msg": msg}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("필수 레이블이 없습니다: %v", [missing])
        }
```

### Constraint 예시 (정책 적용)

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: pod-must-have-team-label
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces: ["production"]
  parameters:
    labels: ["team", "app"]
```

이 설정으로 `production` 네임스페이스의 모든 파드는 `team`과 `app` 레이블이 없으면 배포가 거부된다.

---

## Gatekeeper Audit 기능

Gatekeeper는 새로 들어오는 요청만 검증하는 게 아니라, 이미 클러스터에 존재하는 리소스가 정책을 위반하는지도 주기적으로 감사(Audit)한다.

```bash
# 정책 위반 현황 확인
kubectl get constraint pod-must-have-team-label -o yaml

# 위반 목록은 .status.violations 필드에 표시됨
```

```yaml
status:
  violations:
    - enforcementAction: deny
      kind: Pod
      name: legacy-pod
      namespace: production
      message: "필수 레이블이 없습니다: {\"team\"}"
```

---

## enforcementAction 설정

실제 운영 환경에서 새 정책을 바로 `deny`로 적용하면 기존 워크로드에 영향을 줄 수 있다. 점진적 적용을 위한 옵션이 있다.

| enforcementAction | 동작 |
|-------------------|------|
| `deny` | 정책 위반 시 요청 거부 |
| `warn` | 경고 반환, 요청은 허용 (k8s 1.19+) |
| `dryrun` | 감사만 수행, 실제 거부 없음 |

```yaml
spec:
  enforcementAction: warn   # 먼저 warn으로 영향도 파악
```

---

## Admission Controller 설계 시 고려사항

### failurePolicy

웹훅 서버가 응답하지 않을 때의 동작을 결정한다.

```yaml
failurePolicy: Fail    # 웹훅 실패 → 요청 거부 (보안 우선)
failurePolicy: Ignore  # 웹훅 실패 → 요청 허용 (가용성 우선)
```

프로덕션에서는 `Fail`이 기본이지만, 웹훅 서버 자체의 고가용성(HA)을 반드시 확보해야 한다.

### namespaceSelector

웹훅 적용 대상 네임스페이스를 레이블로 필터링한다. 시스템 네임스페이스(`kube-system`)는 정책 적용에서 제외하는 경우가 많다.

```yaml
namespaceSelector:
  matchExpressions:
    - key: admission-control
      operator: In
      values: ["enabled"]
```

### 성능 영향

모든 API 요청이 웹훅을 거치므로 웹훅 응답 지연이 클러스터 전체에 영향을 준다. `timeoutSeconds`를 합리적으로 설정하고 웹훅 서버 응답 시간을 모니터링해야 한다.

---

## OPA vs Kyverno 비교

Kyverno는 Kubernetes 전용으로 설계된 또 다른 정책 엔진이다.

| 항목 | OPA/Gatekeeper | Kyverno |
|------|----------------|---------|
| 정책 언어 | Rego (범용, 학습 곡선 높음) | YAML (k8s 네이티브) |
| 적용 범위 | k8s 외 다양한 환경 가능 | k8s 전용 |
| Mutate 지원 | 제한적 | 강력한 Mutate 지원 |
| Generate 지원 | 미지원 | ConfigMap/Secret 자동 생성 지원 |
| 도입 난이도 | 상대적으로 높음 | 상대적으로 낮음 |

단순한 k8s 정책 관리라면 Kyverno가 진입 장벽이 낮고, 멀티 플랫폼 정책 통합이 필요하다면 OPA가 적합하다.

---

## 정리

- Admission Controller는 k8s 리소스가 etcd에 저장되기 전 요청을 가로채 검증/변경한다
- Webhook 방식으로 외부 정책 서버와 연동할 수 있다
- OPA는 Rego 언어로 정책을 표현하는 범용 엔진이며, Gatekeeper를 통해 k8s와 통합된다
- ConstraintTemplate → Constraint 구조로 정책을 선언적으로 관리한다
- Audit 기능으로 기존 리소스의 정책 위반도 감지할 수 있다
- `enforcementAction: warn/dryrun`으로 점진적 정책 적용이 가능하다
