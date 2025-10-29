---
layout: post
title: "[Daily morning study] Docker와 Kubernetes(K8s)의 명확한 역할 차이"
description: >
  #daily morning study
category: 
    - dms
    - devopsandcloud
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---
# Docker와 Kubernetes(K8s)의 명확한 역할 차이

## 1. Docker란?

Docker는 컨테이너 기술을 이용하여 소프트웨어를 격리된 환경에서 실행할 수 있게 해주는 플랫폼입니다. Docker를 통해 개발자는 애플리케이션과 그에 필요한 모든 종속성을 컨테이너라는 단위로 포장할 수 있으며, 이는 어디서나 일관되게 실행될 수 있습니다. 

### 1.1 Docker의 주요 개념

- **이미지 (Image)**: 애플리케이션과 그에 필요한 라이브러리, 종속성을 포함한 읽기 전용 템플릿.
  
- **컨테이너 (Container)**: 이미지를 실행한 인스턴스이며, 격리된 환경에서 애플리케이션이 실행됩니다.

- **Dockerfile**: 이미지를 생성하기 위한 설정 파일로, 필요한 소프트웨어와 구성 요소를 정의합니다.

```dockerfile
# 예시 Dockerfile
FROM ubuntu:20.04

RUN apt-get update && apt-get install -y python3

COPY . /app
WORKDIR /app

CMD ["python3", "app.py"]
```

## 2. Kubernetes(K8s)란?

Kubernetes는 컨테이너화된 애플리케이션을 자동으로 배포하고, 확장하며, 관리할 수 있는 오픈 소스 플랫폼입니다. Kubernetes는 다수의 컨테이너, 특히 Docker 컨테이너를 관리하며, 이를 통해 클라우드 환경에서 애플리케이션을 운영하는 데 강력한 도구가 됩니다.

### 2.1 Kubernetes의 주요 개념

- **Pod**: Kubernetes에서 가장 작은 배포 단위로, 하나 이상의 컨테이너를 포함합니다.

- **Service**: Pods 간의 네트워크 통신을 위한 안정적인 접근 방법을 제공하는 추상화 계층입니다.

- **Deployment**: 애플리케이션을 배포하고 업데이트하는 관리 객체입니다. 

```yaml
# 예시 Deployment 파일
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
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
      - name: my-app
        image: my-app:latest
        ports:
        - containerPort: 80
```

## 3. Docker와 Kubernetes의 역할 차이

| 요소            | Docker                                           | Kubernetes                                    |
|----------------|------------------------------------------------|----------------------------------------------|
| 목적           | 애플리케이션을 컨테이너화하여 배포 및 실행    | 컨테이너화된 애플리케이션을 관리 및 오케스트레이션 |
| 구성 요소       | 이미지, 컨테이너, Dockerfile                   | Pod, Service, Deployment 등                     |
| 역할           | 실행 환경 제공                                 | 배포, 스케일링, 로드 밸런싱, 모니터링        |
| 상호 관계      | Kubernetes는 Docker와 함께 사용하는 경우가 많음 | Docker로 만들어진 컨테이너를 관리하기 위해 사용 |

## 4. Docker와 Kubernetes의 사용 사례

### 4.1 Docker 사용 사례

- 개발 환경 설정: 팀원 간 동일한 환경에서 개발할 수 있도록 공유.
- CI/CD 파이프라인 구축: 테스트, 빌드, 배포 과정에서 컨테이너 활용.

### 4.2 Kubernetes 사용 사례

- 대규모 애플리케이션 운영: 여러 개의 컨테이너를 관리하고 확장.
- 업데이트 및 롤백: 애플리케이션을 중단 없이 업데이트 가능.

## 5. 결론

Docker와 Kubernetes는 서로 다른 목적과 기능을 가지고 있으며, 현대 애플리케이션 배포 및 관리를 효과적으로 지원합니다. Docker는 개발과 컨테이너화에 중점을 둔다면, Kubernetes는 이러한 컨테이너를 관리하고 오케스트레이션하는 데 필요한 강력한 도구입니다. 이 두 기술을 함께 활용하면 더 효율적이고 안정적인 배포 환경을 구축할 수 있습니다.
