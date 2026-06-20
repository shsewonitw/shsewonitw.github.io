---
layout: post
title: "[Daily morning study] Kubernetes 네트워킹 - Service, Ingress, NetworkPolicy"
description: >
  #daily morning study
category: 
    - dms
    - dms-devops
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## Kubernetes 네트워킹 기초

Kubernetes 클러스터 안에서 Pod끼리, 그리고 외부에서 Pod로의 통신은 여러 계층의 네트워킹 추상화로 이루어진다. 핵심은 세 가지다: **Service**, **Ingress**, **NetworkPolicy**.

---

## Service

Pod는 재생성될 때마다 IP가 바뀐다. Service는 Pod 집합 앞에서 고정된 DNS 이름과 IP를 제공하는 추상 계층이다. 레이블 셀렉터로 대상 Pod를 선택한다.

### Service 종류

| 타입 | 설명 | 외부 접근 |
|------|------|-----------|
| ClusterIP | 클러스터 내부에서만 접근 가능 (기본값) | 불가 |
| NodePort | 각 노드의 특정 포트를 열어 외부 접근 허용 (30000-32767) | 가능 |
| LoadBalancer | 클라우드 프로바이더의 LB를 프로비저닝 | 가능 |
| ExternalName | 외부 DNS 이름으로 CNAME 리다이렉트 | - |

#### ClusterIP (기본값)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
```

클러스터 내부 다른 Pod에서 `my-service:80`으로 접근할 수 있다. kube-dns가 `my-service.namespace.svc.cluster.local`로 이름을 해석해 준다.

#### NodePort

```yaml
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080
```

`<노드-IP>:30080`으로 외부에서 접근 가능하다. 개발/테스트 환경에서 사용하고 프로덕션에서는 LoadBalancer나 Ingress를 쓰는 게 일반적이다.

#### LoadBalancer

```yaml
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8080
```

AWS ELB, GCP CLB 같은 클라우드 제공 로드밸런서가 자동으로 생성된다. `kubectl get svc`로 EXTERNAL-IP를 확인할 수 있다. 서비스마다 LB가 하나씩 생성되므로 비용이 발생한다.

### Headless Service

ClusterIP를 `None`으로 설정하면 프록시 없이 Pod IP를 직접 DNS 레코드로 반환한다. StatefulSet에서 각 Pod에 개별 DNS로 접근해야 할 때 (예: Kafka, Cassandra 클러스터) 사용한다.

```yaml
spec:
  clusterIP: None
  selector:
    app: kafka
```

---

## Ingress

Service의 LoadBalancer 타입은 서비스마다 외부 LB를 생성해 비용이 크다. **Ingress**는 클러스터 진입점을 하나로 통합하고 호스트 이름이나 URL 경로 기반으로 트래픽을 내부 Service로 라우팅한다.

```
인터넷 → Ingress Controller (nginx, traefik, etc.)
              ↓          ↓            ↓
         api-service  web-service  admin-service
```

### Ingress 리소스

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /v1
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
    - host: www.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
  tls:
    - hosts:
        - api.example.com
        - www.example.com
      secretName: tls-secret
```

Ingress 리소스만으로는 동작하지 않는다. 클러스터에 **Ingress Controller**가 별도로 배포돼 있어야 한다. 일반적으로 `ingress-nginx`나 Traefik을 사용한다.

### pathType 종류

| 값 | 설명 |
|----|------|
| Exact | 경로가 정확히 일치해야 함 |
| Prefix | 해당 prefix로 시작하는 경로 모두 매칭 |
| ImplementationSpecific | Ingress Controller가 정의 |

---

## NetworkPolicy

기본적으로 Kubernetes 클러스터 내 모든 Pod는 서로 통신할 수 있다. **NetworkPolicy**는 Pod 간 네트워크 트래픽을 허용/차단하는 방화벽 규칙이다.

NetworkPolicy가 적용되려면 CNI 플러그인이 NetworkPolicy를 지원해야 한다 (Calico, Cilium, Weave Net 등). Flannel은 기본적으로 지원하지 않는다.

### 기본 격리 정책

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

이 정책을 적용하면 `production` 네임스페이스의 모든 Pod에 대한 인바운드/아웃바운드 트래픽이 모두 차단된다. 이후 필요한 통신만 명시적으로 허용한다.

### 특정 Pod 간 통신 허용

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-from-frontend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```

### 네임스페이스 간 허용

```yaml
ingress:
  - from:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: monitoring
        podSelector:
          matchLabels:
            app: prometheus
```

`namespaceSelector`와 `podSelector`를 같은 `-` 항목 아래 두면 AND 조건이다. 별도 `-` 항목으로 분리하면 OR 조건이 된다.

---

## kube-proxy의 역할

Service 트래픽이 실제 Pod로 전달되는 방식은 각 노드에서 실행 중인 **kube-proxy**가 처리한다. kube-proxy는 iptables 또는 IPVS 규칙을 설정해 Service IP로 들어오는 패킷을 실제 Pod IP로 NAT한다.

```
클라이언트 → ClusterIP:80 → iptables(kube-proxy) → PodIP:8080
```

IPVS 모드는 iptables보다 확장성이 좋아 Pod 수가 많은 대규모 클러스터에서 권장된다.

---

## DNS 동작 방식

Kubernetes는 CoreDNS를 기본 클러스터 DNS로 사용한다. 서비스 이름으로 접근할 때 다음 순서로 이름을 해석한다.

```
my-service
→ my-service.default.svc.cluster.local  (같은 네임스페이스)
→ my-service.other-ns.svc.cluster.local
→ ...
```

Pod의 `/etc/resolv.conf`에 클러스터 도메인(`cluster.local`)이 search 항목으로 등록되어 있어 짧은 이름으로도 접근이 가능하다.

---

## 정리

| 개념 | 역할 |
|------|------|
| ClusterIP | 클러스터 내부 서비스 노출 (기본값) |
| NodePort | 노드 포트를 통한 외부 노출 |
| LoadBalancer | 클라우드 LB를 통한 외부 노출 |
| Ingress | 단일 진입점에서 URL/호스트 기반 라우팅 |
| NetworkPolicy | Pod 간 트래픽 화이트리스트 방화벽 |
| kube-proxy | Service IP → Pod IP 패킷 변환 |

- Service는 Pod 집합에 안정적인 접근점을 제공하고, Ingress는 HTTP(S) 트래픽을 경제적으로 외부에 노출한다.
- NetworkPolicy는 기본값이 전체 허용이므로 민감한 환경에서는 `default-deny` 정책을 먼저 적용하고 필요한 통신만 열어야 한다.
