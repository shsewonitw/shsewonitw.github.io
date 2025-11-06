---
layout: post
title: "[Daily morning study] Docker와 Kubernetes(K8s)의 명확한 역할 차이"
description: >
  #daily morning study
category: 
    - dms
    - -devops
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---
# Docker와 Kubernetes(K8s)의 명확한 역할 차이

Docker와 Kubernetes는 모두 컨테이너화된 애플리케이션을 관리하는 데 도움을 주지만, 그 역할은 상당히 다릅니다. 이 문서는 두 기술 간의 주요 차이를 설명하고, 각 기술이 어떻게 작동하는지에 대해 알아보겠습니다.

---

## 1. Docker란 무엇인가?

### 1.1 정의
Docker는 애플리케이션을 컨테이너라는 경량의 가상 환경에서 실행할 수 있도록 해주는 플랫폼입니다. 컨테이너는 애플리케이션과 그 애플리케이션이 필요로 하는 모든 종속성을 포함합니다.

### 1.2 주요 특징
- **경량성**: 컨테이너는 VM보다 훨씬 가볍고 빠르며, 여러 개의 컨테이너가 단일 호스트 OS에서 동시에 실행될 수 있습니다.
- **이식성**: 컨테이너는 어떠한 환경에서도 일관되게 실행될 수 있어, 개발자들이 로컬에서 테스트한 내용을 실제 프로덕션 환경에서도 똑같이 사용할 수 있습니다.
- **버전 관리**: Docker 이미지를 통해 버전 관리가 가능하여, 원하는 레퍼런스 상태로 쉽게 되돌아갈 수 있습니다.

### 1.3 기본 사용 방법
Docker를 사용하여 간단한 웹 애플리케이션 컨테이너를 만들고 실행하는 예시는 다음과 같습니다. 

```bash
# Dockerfile 예시
FROM nginx:alpine

COPY ./my_website /usr/share/nginx/html

# 컨테이너 이미지 빌드
docker build -t my-nginx-image .

# 컨테이너 실행
docker run -d -p 80:80 my-nginx-image
```

---

## 2. Kubernetes(K8s)란 무엇인가?

### 2.1 정의
Kubernetes는 컨테이너화된 애플리케이션을 배포, 스케일링 및 운영하기 위한 오케스트레이션 플랫폼입니다. Kubernetes는 여러 대의 호스트에서 수천 개의 컨테이너를 효율적으로 관리할 수 있게 해줍니다.

### 2.2 주요 특징
- **자동화된 배포 및 롤아웃**: Kubernetes는 새로운 기능을효율적으로 배포하고 기존 애플리케이션의 문제를 최소화하도록 설계되었습니다.
- **스케일링**: Kubernetes는 애플리케이션의 트래픽 변화에 따라 자동으로 컨테이너의 수를 조절합니다.
- **셀프-Healing**: 장애가 발생한 컨테이너를 자동으로 재시작하거나 대체할 수 있는 기능이 있습니다.

### 2.3 기본 사용 방법
Kubernetes에서 간단한 Deployment를 생성하는 YAML 파일 예시는 다음과 같습니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
```

위의 YAML 파일을 사용하여 Deployment를 생성할 수 있습니다.

```bash
kubectl apply -f nginx-deployment.yaml
```

---

## 3. Docker와 Kubernetes의 역할 차이

| 역할          | Docker                                  | Kubernetes                        |
|--------------|----------------------------------------|----------------------------------|
| 기본 기능     | 컨테이너 생성 및 관리                    | 컨테이너 오케스트레이션          |
| 스케일링      | 컨테이너 개별 관리                       | 여러 컨테이너의 자동 스케일링    |
| 배포          | 단일 컨테이너 배포                      | 멀티 컨테이너 및 서비스 배포     |
| 로드 밸런싱   | 직접적인 로드 밸런싱 없음              | 내장된 로드 밸런싱 기능 제공     |
| 배포 환경     | 개발 및 테스트 환경에 최적화            | 프로덕션 환경에서의 안정성 보장      |

---

## 4. 결론

Docker와 Kubernetes는 서로 보완적인 역할을 하며, Docker는 컨테이너를 만들고 실행하는 데 집중하고, Kubernetes는 이러한 컨테이너를 관리하고 운영하는 데 중점을 둡니다. 따라서 이 두 기술을 함께 사용하면 애플리케이션을 보다 효율적이고 안정적으로 운영할 수 있습니다. 

이 문서를 통해 Docker와 Kubernetes의 역할 차이를 이해하는 데 도움이 되었기를 바랍니다.
