---
layout: post
title: "[Daily morning study] AWS ECS vs EKS 컨테이너 오케스트레이션 비교"
description: >
  #daily morning study
category: 
    - dms
    - dms-cloud
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## AWS 컨테이너 오케스트레이션: ECS vs EKS

컨테이너로 서비스를 운영할 때 가장 먼저 부딪히는 선택지가 ECS(Elastic Container Service)냐 EKS(Elastic Kubernetes Service)냐다. 둘 다 컨테이너를 실행하고 관리하는 플랫폼이지만 철학과 운영 방식이 완전히 다르다.

---

## ECS (Elastic Container Service)

AWS가 자체 개발한 컨테이너 오케스트레이터다. Kubernetes 없이 AWS 생태계 안에서 컨테이너를 관리한다.

### 핵심 개념

| 개념 | 설명 |
|------|------|
| **Cluster** | 컨테이너를 실행할 인프라 그룹 |
| **Task Definition** | 컨테이너 이미지, CPU/메모리, 환경변수 등을 정의한 설계도 |
| **Task** | Task Definition을 기반으로 실행된 컨테이너 인스턴스 |
| **Service** | Task를 원하는 개수만큼 유지시키는 관리 단위 |

### 실행 타입: EC2 vs Fargate

ECS에서 컨테이너를 돌리는 방식은 두 가지다.

**EC2 타입**: 직접 EC2 인스턴스를 띄우고 그 위에서 컨테이너를 실행한다. 인스턴스 관리를 직접 해야 하지만 비용 최적화 여지가 크다.

**Fargate 타입**: 서버리스 방식으로 EC2 인스턴스 없이 컨테이너만 정의하면 AWS가 인프라를 알아서 관리한다. 인프라 관리 부담이 없는 대신 비용이 더 비싸고 제어 범위가 좁다.

```json
// Task Definition 예시 (일부)
{
  "family": "my-app",
  "containerDefinitions": [
    {
      "name": "web",
      "image": "my-ecr-repo/web:latest",
      "cpu": 256,
      "memory": 512,
      "portMappings": [
        {
          "containerPort": 80,
          "protocol": "tcp"
        }
      ],
      "environment": [
        { "name": "NODE_ENV", "value": "production" }
      ]
    }
  ],
  "requiresCompatibilities": ["FARGATE"],
  "networkMode": "awsvpc",
  "cpu": "256",
  "memory": "512"
}
```

### ECS의 특징

- AWS 콘솔, CLI, CloudFormation, CDK로 쉽게 관리
- IAM, ALB, CloudWatch, ECR과 네이티브 통합
- 학습 곡선이 낮음
- 쿠버네티스 지식 없어도 운영 가능

---

## EKS (Elastic Kubernetes Service)

AWS에서 관리형 Kubernetes 클러스터를 제공하는 서비스다. 쿠버네티스 컨트롤 플레인을 AWS가 관리해주고, 워커 노드만 사용자가 운영한다.

### 핵심 개념

Kubernetes의 표준 객체를 그대로 사용한다.

| 개념 | 설명 |
|------|------|
| **Pod** | 컨테이너의 최소 실행 단위 (1개 이상의 컨테이너 묶음) |
| **Deployment** | Pod 개수 관리, 롤링 업데이트 담당 |
| **Service** | Pod에 대한 네트워크 접근 추상화 |
| **Ingress** | 외부 HTTP/HTTPS 트래픽을 Service로 라우팅 |
| **Namespace** | 클러스터 내 논리적 분리 단위 |

### 노드 그룹 타입

EKS도 ECS처럼 노드 관리 방식을 선택할 수 있다.

- **Managed Node Group**: EC2 인스턴스를 AWS가 생성/업데이트/삭제 관리
- **Self-managed Node**: 사용자가 EC2를 직접 프로비저닝
- **Fargate Profile**: 서버리스 방식으로 Pod 실행 (ECS Fargate와 유사)

```yaml
# Kubernetes Deployment 예시
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: web
          image: my-ecr-repo/web:latest
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "500m"
              memory: "1Gi"
          env:
            - name: NODE_ENV
              value: production
```

### EKS의 특징

- 표준 Kubernetes API 완벽 호환 → 멀티 클라우드 이식성
- Helm, Kustomize, ArgoCD 등 쿠버네티스 생태계 도구 활용 가능
- HPA(수평 파드 자동 스케일링), VPA(수직), KEDA(이벤트 기반) 등 강력한 스케일링
- 컨트롤 플레인 비용 시간당 약 $0.10 (ECS는 무료)
- 학습 곡선이 높음

---

## ECS vs EKS 비교 요약

| 항목 | ECS | EKS |
|------|-----|-----|
| 오케스트레이터 | AWS 자체 | Kubernetes |
| 학습 곡선 | 낮음 | 높음 |
| 컨트롤 플레인 비용 | 무료 | 시간당 $0.10 |
| 멀티 클라우드 이식성 | 어려움 (AWS 종속) | 쉬움 (K8s 표준) |
| 생태계 도구 | AWS 도구 중심 | 광범위한 K8s 생태계 |
| 서버리스 지원 | Fargate | Fargate Profile |
| 네트워킹 복잡도 | 낮음 | 높음 (CNI, Service Mesh 등) |
| 세밀한 제어 | 제한적 | 매우 높음 |
| AWS 서비스 통합 | 네이티브 통합 | 추가 설정 필요 |

---

## 언제 뭘 선택해야 할까

**ECS를 선택하는 경우**

- 팀이 쿠버네티스를 처음 접하거나 러닝커브를 최소화해야 할 때
- AWS 환경에 완전히 종속되어도 괜찮은 경우
- 빠르게 컨테이너 기반 서비스를 런칭해야 할 때
- 간단한 마이크로서비스 구성이거나 운영 규모가 크지 않을 때
- Fargate로 인프라 관리를 최소화하고 싶을 때

**EKS를 선택하는 경우**

- 이미 쿠버네티스를 쓰고 있거나 팀에 K8s 경험자가 있을 때
- 멀티 클라우드 또는 하이브리드 클라우드를 고려할 때
- Helm 차트, ArgoCD, Istio 같은 K8s 에코시스템을 활용해야 할 때
- 세밀한 네트워크 정책, 리소스 제어, 커스텀 스케줄링이 필요할 때
- 대규모 서비스로 확장될 것을 염두에 두고 있을 때

---

## 네트워킹 모델 차이

두 서비스의 네트워킹은 구조가 다르다.

**ECS (awsvpc 모드)**

각 Task가 독립적인 ENI(Elastic Network Interface)를 가진다. 컨테이너 단위로 Security Group을 적용할 수 있어 간단하고 직관적이다.

**EKS**

Kubernetes CNI 플러그인(AWS VPC CNI)을 사용한다. Pod마다 VPC IP를 직접 할당받는다. Service 타입으로 ClusterIP, NodePort, LoadBalancer를 골라 쓰고, Ingress Controller로 외부 트래픽을 라우팅한다.

```
# EKS 트래픽 흐름
Internet → ALB (Ingress) → Service (ClusterIP) → Pod
```

---

## 비용 고려 사항

ECS는 컨트롤 플레인 비용이 없다. EC2 인스턴스나 Fargate Task 비용만 낸다.

EKS는 쿠버네티스 클러스터 자체 비용으로 클러스터당 시간당 $0.10이 추가된다. 한 달이면 약 $72다. 클러스터를 여러 개 운영하면 비용이 선형으로 증가한다.

Fargate를 쓰는 경우 ECS Fargate와 EKS Fargate의 단가는 동일하다. 차이는 클러스터 관리 비용뿐이다.

---

## AWS 생태계 통합

ECS는 AWS 서비스와의 통합이 더 자연스럽다.

- **ALB**: Target Group에 Task를 자동 등록/해제
- **CloudWatch**: 컨테이너 로그와 메트릭 자동 수집
- **IAM**: Task Role로 컨테이너에 권한 부여 (세밀한 서비스별 권한 설정)
- **Systems Manager Parameter Store**: 환경변수로 시크릿 주입

EKS도 같은 기능들을 사용할 수 있지만 별도 설정(IRSA, AWS Load Balancer Controller, Fluent Bit DaemonSet 등)이 필요하다.

---

## 정리

- **단순함 우선**: ECS + Fargate 조합이 가장 빠르고 관리 부담이 적다
- **유연성/표준**: EKS가 더 강력하지만 운영 복잡도가 올라간다
- **팀 역량**: K8s 전문가가 없으면 ECS가 현실적인 선택
- **장기 방향**: 멀티 클라우드나 K8s 생태계를 적극 활용할 계획이면 EKS에 투자할 가치가 있다
