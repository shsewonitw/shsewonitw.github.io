---
layout: post
title: "[Daily morning study] GitOps와 ArgoCD 개념"
description: >
  #daily morning study
category: 
    - dms
    - dms-devops
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## GitOps란

GitOps는 Git을 인프라와 애플리케이션 배포의 **단일 진실 소스(Single Source of Truth)**로 삼는 운영 방식이다. 쉽게 말하면, "Git에 있는 것이 곧 실제 시스템의 상태여야 한다"는 원칙이다.

2017년 Weaveworks가 처음 제안한 개념으로, 다음 네 가지 원칙을 기반으로 한다.

| 원칙 | 설명 |
|------|------|
| 선언적(Declarative) | 인프라와 앱 상태를 코드로 선언적으로 기술 |
| 버전 관리 | 모든 변경 이력이 Git에 기록됨 |
| 자동 적용 | Git 변경 사항이 자동으로 시스템에 반영 |
| 지속적 동기화 | 실제 시스템 상태와 Git 상태를 지속적으로 비교·교정 |

---

## GitOps vs 기존 CI/CD

기존 CI/CD 파이프라인은 보통 **Push** 방식이다. 코드가 푸시되면 CI 서버가 빌드하고, 직접 클러스터에 배포 명령을 실행한다.

```
[Git Push] → [CI 서버: 빌드/테스트] → [kubectl apply] → [k8s 클러스터]
```

GitOps는 **Pull** 방식으로 동작한다. 클러스터 내부의 에이전트가 Git을 주기적으로 감시하다가, 차이(diff)가 생기면 스스로 동기화한다.

```
[Git Push] → [CI 서버: 빌드/이미지 푸시] → [Git 매니페스트 업데이트]
                                                        ↓
                                          [에이전트가 diff 감지 후 Pull & Apply]
```

### 핵심 차이점

- 기존 방식: CI 서버가 클러스터에 직접 접근(Push)해서 배포
- GitOps: 클러스터 안의 에이전트가 Git을 Pull해서 스스로 반영

이 차이 덕분에 클러스터 외부에서 직접 접근하는 자격증명을 CI 서버에 줄 필요가 없어 보안이 향상된다.

---

## ArgoCD란

ArgoCD는 Kubernetes를 위한 GitOps 기반 **지속적 배포(Continuous Delivery)** 도구다.

Git 리포지토리에 정의된 Kubernetes 매니페스트(YAML)를 실제 클러스터 상태와 지속적으로 비교해서, 차이가 생기면 자동 또는 수동으로 동기화한다.

### 주요 특징

- **선언적 설정**: 배포할 앱의 소스(Git URL, 경로)와 대상 클러스터를 ArgoCD Application 리소스로 선언
- **실시간 상태 확인**: 웹 UI에서 앱의 현재 배포 상태, 리소스 트리를 시각적으로 확인 가능
- **자동/수동 동기화**: 자동 동기화 설정 시 Git 변경 사항이 바로 클러스터에 반영
- **롤백**: Git 히스토리가 곧 배포 이력이므로, 원하는 커밋으로 간단히 롤백 가능
- **멀티 클러스터**: 단일 ArgoCD 인스턴스로 여러 Kubernetes 클러스터를 관리할 수 있음

---

## ArgoCD 핵심 개념

### Application

ArgoCD에서 배포 단위를 **Application**이라 부른다. 다음 정보를 담는다.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/my-org/my-app-config
    targetRevision: HEAD
    path: k8s/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true       # Git에서 삭제된 리소스는 클러스터에서도 삭제
      selfHeal: true    # 클러스터 상태가 수동으로 바뀌어도 Git 상태로 복원
```

### Sync 상태

ArgoCD는 Application의 상태를 항상 두 가지로 표시한다.

| 상태 | 설명 |
|------|------|
| **Synced** | Git 상태 = 클러스터 실제 상태 (정상) |
| **OutOfSync** | Git 상태 ≠ 클러스터 실제 상태 (동기화 필요) |

### Health 상태

| 상태 | 설명 |
|------|------|
| Healthy | 모든 리소스가 정상 동작 중 |
| Degraded | 일부 리소스가 오류 상태 |
| Progressing | 배포 진행 중 |
| Missing | 리소스가 클러스터에 존재하지 않음 |

---

## ArgoCD 동작 흐름

```
1. 개발자가 애플리케이션 코드 변경 → CI에서 빌드 후 Docker 이미지 푸시
2. CI 또는 개발자가 Git 매니페스트 리포지토리의 image 태그 업데이트
3. ArgoCD가 주기적으로(기본 3분) 또는 webhook으로 Git 변경 감지
4. Git 상태와 클러스터 상태 비교 → OutOfSync 감지
5. 자동 동기화 설정 시 자동으로 kubectl apply, 수동이면 대기
6. 배포 완료 후 Synced + Healthy 상태로 전환
```

---

## Kustomize / Helm 연동

ArgoCD는 raw YAML뿐 아니라 **Kustomize**와 **Helm**을 기본 지원한다.

**Helm 예시**

```yaml
source:
  repoURL: https://charts.bitnami.com/bitnami
  chart: redis
  targetRevision: 17.x.x
  helm:
    values: |
      auth:
        enabled: false
      replica:
        replicaCount: 2
```

**Kustomize 예시**

```yaml
source:
  repoURL: https://github.com/my-org/my-app-config
  path: k8s/overlays/staging
  targetRevision: HEAD
  kustomize:
    images:
      - my-app=my-registry/my-app:v1.2.3
```

---

## ArgoCD 설치 (간단 요약)

```bash
# argocd 네임스페이스 생성 후 설치
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 초기 admin 비밀번호 확인
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# ArgoCD CLI 로그인
argocd login <ARGOCD_SERVER>
```

---

## GitOps 리포지토리 구조 패턴

GitOps를 도입할 때 리포지토리를 어떻게 구성할지가 중요하다.

### 모노레포 방식

```
my-gitops-repo/
├── apps/
│   ├── frontend/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── backend/
│       ├── deployment.yaml
│       └── service.yaml
└── infra/
    ├── namespaces.yaml
    └── ingress.yaml
```

### 멀티레포 방식

```
app-source-repo/      ← 애플리케이션 소스 코드 (CI가 이미지 빌드)
app-config-repo/      ← Kubernetes 매니페스트만 (ArgoCD가 감시)
```

멀티레포 방식이 더 흔히 사용된다. 소스 코드 변경과 배포 설정 변경을 분리할 수 있고, 누가 언제 배포 설정을 변경했는지 추적이 쉽다.

---

## GitOps의 장점 정리

- **감사 추적(Audit Trail)**: 모든 변경이 Git 커밋으로 남아 "누가, 언제, 무엇을" 바꿨는지 명확
- **롤백 용이**: `git revert` 또는 이전 커밋으로 체크아웃하면 즉시 이전 상태로 복원
- **드리프트 방지**: 누군가 클러스터를 수동으로 건드려도 `selfHeal`이 Git 상태로 되돌림
- **보안 향상**: CI 서버가 클러스터에 직접 접근할 자격증명이 필요 없음
- **협업 친화적**: Pull Request 기반으로 배포 변경을 리뷰하고 승인할 수 있음

---

## 정리

GitOps = "Git이 곧 배포 상태"라는 원칙. ArgoCD는 그 원칙을 Kubernetes에서 구현하는 도구다. 코드 배포뿐 아니라 클러스터 설정, RBAC, 네임스페이스 생성 같은 인프라 변경도 Git을 통해 관리할 수 있어서, 대규모 Kubernetes 운영 환경에서 사실상 표준처럼 쓰이고 있다.
