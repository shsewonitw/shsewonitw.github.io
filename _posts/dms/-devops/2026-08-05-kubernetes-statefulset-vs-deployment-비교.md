---
layout: post
title: "[Daily morning study] Kubernetes StatefulSet vs Deployment 비교"
description: >
  #daily morning study
category: 
    - dms
    - dms-devops
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## Deployment vs StatefulSet — 언제 무엇을 써야 하나

Kubernetes에서 파드(Pod)를 여러 개 실행할 때 보통 Deployment를 쓴다. 그런데 데이터베이스나 메시지 큐처럼 **상태(state)를 가지는 애플리케이션**을 올리려 하면 Deployment만으로는 부족하다는 걸 금방 알게 된다. 이 차이를 정리해 두면 설계할 때 헷갈리지 않는다.

---

## Deployment

Deployment는 **무상태(stateless) 워크로드**를 위해 설계됐다. 핵심 특성은 다음과 같다.

- 파드 이름이 `{deployment-name}-{replicaset-hash}-{random-suffix}` 형태로 무작위 생성됨
- 파드끼리 **동등(interchangeable)**하다 — 어느 파드가 요청을 처리해도 상관없음
- 파드가 죽으면 새로운 이름으로 재생성됨
- 볼륨을 붙여도 파드가 바뀌면 같은 볼륨을 보장하지 않음
- 롤링 업데이트 시 순서 보장 없이 병렬로 교체 가능

```yaml
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
        - name: app
          image: my-app:v2
```

웹 서버, API 서버처럼 요청마다 독립적으로 처리할 수 있는 서비스에 적합하다.

---

## StatefulSet

StatefulSet은 **상태를 가지는 워크로드**를 위해 만들어졌다. Deployment와 결정적으로 다른 점들이 있다.

### 안정적인 네트워크 ID

파드 이름이 `{statefulset-name}-0`, `{statefulset-name}-1`, `{statefulset-name}-2` 처럼 **순서가 있는 고정 이름**으로 생성된다. 파드가 재시작되어도 이름이 유지된다.

```
mysql-0   ← 항상 이 이름
mysql-1
mysql-2
```

### Headless Service와 DNS

StatefulSet은 보통 **Headless Service**(`.spec.clusterIP: None`)와 함께 쓴다. 각 파드에 다음과 같은 DNS 주소가 자동으로 부여된다.

```
{pod-name}.{service-name}.{namespace}.svc.cluster.local

# 예시
mysql-0.mysql.default.svc.cluster.local
mysql-1.mysql.default.svc.cluster.local
```

클라이언트가 특정 파드를 직접 지정해서 접근할 수 있다. Primary DB에만 쓰기를 보내야 하는 경우 이게 필수다.

### 순서가 보장된 배포 및 삭제

- **생성**: 0 → 1 → 2 순서로 하나씩 올라옴 (이전 파드가 Running이 돼야 다음 것 생성)
- **삭제**: 2 → 1 → 0 순서로 역순 삭제
- **업데이트**: 기본적으로 역순(가장 높은 번호부터)으로 롤링 업데이트

이 순서 보장 덕분에 클러스터 멤버십 관리가 안전해진다.

### 안정적인 스토리지 (PVC 보장)

StatefulSet은 `volumeClaimTemplates`을 통해 파드마다 **고유한 PVC(PersistentVolumeClaim)**를 자동으로 만들어 준다.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          volumeMounts:
            - name: data
              mountPath: /var/lib/mysql
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

위 설정이면 `data-mysql-0`, `data-mysql-1`, `data-mysql-2` PVC가 자동 생성된다. 파드가 재스케줄되어도 **같은 PVC에 다시 붙는다**. Deployment에서는 이게 안 된다.

---

## 두 가지를 한눈에 비교

| 항목 | Deployment | StatefulSet |
|------|-----------|-------------|
| 파드 이름 | 랜덤 suffix | 순서 있는 고정 이름 (`-0`, `-1`) |
| 네트워크 ID | 파드 재시작 시 변경 | 재시작해도 동일 |
| 스토리지 | 파드마다 보장 없음 | 파드마다 전용 PVC |
| 배포/삭제 순서 | 순서 없이 병렬 가능 | 순서 보장 |
| 주요 사용처 | 웹서버, API서버 | DB, 메시지 큐, ZooKeeper |

---

## 실제로 어떤 상황에서 쓰나

**StatefulSet을 써야 하는 경우**

- MySQL, PostgreSQL Replication (Primary/Replica 구분이 필요)
- Kafka, RabbitMQ (브로커 ID가 고정이어야 클러스터가 유지됨)
- Elasticsearch, Cassandra (클러스터 노드 간 통신 시 안정적인 주소 필요)
- ZooKeeper (쿼럼 구성을 위해 각 노드 식별 필수)

**Deployment로 충분한 경우**

- Nginx, Apache 등 웹 서버
- 외부 DB를 쓰는 REST API 서버
- 스테이트리스 워커 프로세스

---

## StatefulSet의 주의점

**PVC는 자동으로 삭제되지 않는다.** StatefulSet을 `kubectl delete` 해도 PVC는 남는다. 데이터 보호를 위한 의도적인 설계인데, 직접 PVC를 삭제해야 완전히 정리된다.

```bash
# StatefulSet 삭제 후에도 PVC가 남아 있음
kubectl get pvc
# data-mysql-0   Bound   ...
# data-mysql-1   Bound   ...
# data-mysql-2   Bound   ...

# 직접 삭제해야 함
kubectl delete pvc data-mysql-0 data-mysql-1 data-mysql-2
```

**스케일 다운 시 데이터 손실 위험**

replicas를 3 → 1로 줄이면 `mysql-1`, `mysql-2` 파드가 삭제되지만 PVC는 남는다. 다시 스케일 업하면 같은 PVC에 다시 연결된다. 하지만 애플리케이션 레벨에서 클러스터 멤버 제거 처리를 먼저 해줘야 하는 경우가 있으니 주의해야 한다.

---

## 정리

Deployment와 StatefulSet의 본질적인 차이는 **파드의 정체성(identity)을 보장하느냐 아니냐**다.

- 어떤 파드가 요청을 처리해도 상관없다 → **Deployment**
- 각 파드가 고유한 역할을 맡고, 재시작해도 같은 이름·같은 데이터를 유지해야 한다 → **StatefulSet**

데이터베이스나 분산 시스템을 Kubernetes 위에 올릴 때 StatefulSet을 이해하지 않으면 클러스터가 제대로 동작하지 않는 상황을 만나게 된다. 개념을 잡아두면 Helm Chart를 읽을 때도 설계 의도가 바로 보인다.
