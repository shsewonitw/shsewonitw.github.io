---
layout: post
title: "[Daily morning study] Kubernetes RBAC와 Service Account 보안 설정"
description: >
  #daily morning study
category: 
    - dms
    - dms-devops
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## Kubernetes RBAC와 Service Account 보안 설정

### RBAC이란?

RBAC(Role-Based Access Control)은 **역할(Role) 기반으로 리소스에 대한 접근 권한을 관리**하는 메커니즘이다. Kubernetes에서는 1.8버전부터 안정적으로 지원되며, 현재는 클러스터 보안 설정에서 가장 핵심적인 요소다.

기본 개념은 간단하다.

- **누가(Subject)** — User, Group, Service Account
- **무엇을(Resource)** — Pod, Deployment, Secret, ConfigMap 등
- **어떻게(Verb)** — get, list, watch, create, update, patch, delete

이 세 가지를 조합해서 권한을 정의한다.

---

### 핵심 오브젝트 4가지

Kubernetes RBAC는 4가지 오브젝트로 구성된다.

| 오브젝트 | 범위 | 설명 |
|---------|------|------|
| Role | 네임스페이스 | 특정 네임스페이스 내의 리소스에 대한 권한 정의 |
| ClusterRole | 클러스터 전체 | 모든 네임스페이스 또는 비네임스페이스 리소스(노드 등)에 대한 권한 정의 |
| RoleBinding | 네임스페이스 | Role 또는 ClusterRole을 특정 주체에 바인딩 |
| ClusterRoleBinding | 클러스터 전체 | ClusterRole을 클러스터 전체 범위로 특정 주체에 바인딩 |

---

### Role과 ClusterRole 작성법

**Role 예시 — 특정 네임스페이스 내에서 Pod를 읽을 수 있는 권한:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: pod-reader
rules:
- apiGroups: [""]        # "" 는 core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

**ClusterRole 예시 — 클러스터 전체에서 Node 정보를 읽을 수 있는 권한:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
```

`apiGroups`는 Kubernetes API 그룹을 의미한다. `""` 는 core 그룹(Pod, Service, ConfigMap 등), `apps`는 Deployment, StatefulSet 등, `batch`는 Job, CronJob 등을 나타낸다.

---

### RoleBinding과 ClusterRoleBinding

Role을 정의했다면 주체(Subject)에 바인딩해야 실제로 권한이 적용된다.

**RoleBinding 예시 — dev 네임스페이스에서 jane이라는 사용자에게 pod-reader 권한 부여:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: dev
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**ClusterRole을 특정 네임스페이스에서만 적용하기:**

ClusterRole은 ClusterRoleBinding 없이도 RoleBinding을 통해 특정 네임스페이스에만 적용할 수 있다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-secrets
  namespace: staging
subjects:
- kind: User
  name: bob
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole   # ClusterRole을 참조하지만 RoleBinding이므로 staging 네임스페이스에만 적용
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

---

### Service Account란?

Service Account는 **파드(Pod) 내의 프로세스가 Kubernetes API 서버와 통신할 때 사용하는 ID**다. 사람이 사용하는 User Account와 달리 애플리케이션/프로세스를 위한 계정이다.

각 네임스페이스에는 기본 Service Account (`default`)가 자동으로 생성되며, 파드를 생성할 때 명시적으로 지정하지 않으면 이 `default` Service Account가 자동으로 마운트된다.

```bash
# 현재 네임스페이스의 Service Account 목록 확인
kubectl get serviceaccounts

# Service Account 생성
kubectl create serviceaccount my-app-sa -n production
```

---

### Service Account에 RBAC 적용하기

애플리케이션에 필요한 최소 권한만 부여하는 것이 보안의 핵심이다 (최소 권한 원칙, Principle of Least Privilege).

**예시 — ConfigMap을 읽는 파드를 위한 Service Account 설정:**

1. Service Account 생성

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: config-reader-sa
  namespace: production
```

2. 필요한 권한을 Role로 정의

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: configmap-reader
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
```

3. RoleBinding으로 Service Account에 Role 연결

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: config-reader-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: config-reader-sa
  namespace: production
roleRef:
  kind: Role
  name: configmap-reader
  apiGroup: rbac.authorization.k8s.io
```

4. 파드에 Service Account 지정

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: production
spec:
  serviceAccountName: config-reader-sa   # 여기서 지정
  containers:
  - name: app
    image: my-app:latest
```

---

### Service Account 토큰 마운트 방식

파드 내에서 Service Account 토큰은 `/var/run/secrets/kubernetes.io/serviceaccount/token` 경로에 자동으로 마운트된다.

Kubernetes 1.21부터는 **Bound Service Account Token**이 기본값이 됐다. 이전 방식(long-lived token)과의 차이점은 다음과 같다.

| 구분 | 기존 방식 | Bound Token (1.21+) |
|------|-----------|---------------------|
| 유효 기간 | 무제한 | 기본 1시간, 갱신됨 |
| 대상 제한 | 없음 | 특정 파드, 오디언스 제한 |
| 보안 | 토큰 탈취 시 영구 유효 | 만료/바인딩으로 위험 최소화 |

**토큰 자동 마운트 비활성화:**

API 서버와 통신할 필요가 없는 파드라면 토큰 마운트를 꺼두는 것이 좋다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-api-access-pod
spec:
  automountServiceAccountToken: false   # 토큰 마운트 비활성화
  containers:
  - name: app
    image: nginx
```

---

### RBAC 디버깅

**권한 확인 — can-i 명령어:**

```bash
# jane이 dev 네임스페이스에서 파드를 생성할 수 있는지 확인
kubectl auth can-i create pods --namespace=dev --as=jane

# 현재 사용자(자신)의 권한 확인
kubectl auth can-i list deployments -n production

# 특정 Service Account의 권한 확인
kubectl auth can-i get secrets \
  --as=system:serviceaccount:production:my-app-sa \
  -n production
```

**권한 목록 조회:**

```bash
# 특정 사용자에게 부여된 RoleBinding 확인
kubectl get rolebindings,clusterrolebindings -A \
  -o wide | grep jane

# 특정 Role의 권한 상세 조회
kubectl describe role pod-reader -n dev
```

---

### 자주 하는 실수와 주의 사항

**1. default Service Account에 과도한 권한 부여**

```yaml
# 절대 하면 안 되는 패턴
kind: ClusterRoleBinding
subjects:
- kind: ServiceAccount
  name: default          # 기본 SA에 cluster-admin 권한
  namespace: default
roleRef:
  kind: ClusterRole
  name: cluster-admin
```

모든 파드가 `default` SA를 사용하므로, 여기에 광범위한 권한을 주면 어느 파드에서든 클러스터 전체를 제어할 수 있게 된다.

**2. wildcard 사용 자제**

```yaml
# 위험한 패턴 — 모든 리소스, 모든 동사
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

실제 필요한 리소스와 동사만 명시적으로 나열하는 것이 원칙이다.

**3. ClusterRoleBinding 범위 오해**

ClusterRoleBinding은 모든 네임스페이스에 걸쳐 적용된다. `namespace`를 지정해도 ClusterRoleBinding 자체의 범위는 변하지 않는다. 특정 네임스페이스에만 제한하고 싶다면 RoleBinding을 사용해야 한다.

---

### 권장 보안 실천 방법 정리

- **최소 권한 원칙**: 애플리케이션에 필요한 최소한의 verb와 resource만 허용
- **전용 Service Account 생성**: `default` SA 재사용 금지, 워크로드별 전용 SA 생성
- **`automountServiceAccountToken: false`**: API 서버 접근이 불필요한 파드에 적용
- **정기적인 권한 감사**: `kubectl auth can-i` 및 `kubectl get rolebindings -A`로 정기 점검
- **네임스페이스 분리**: 환경(dev/staging/production)별 네임스페이스를 분리해 권한 범위를 최소화
