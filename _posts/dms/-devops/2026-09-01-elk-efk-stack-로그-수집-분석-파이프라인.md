---
layout: post
title: "[Daily morning study] ELK/EFK Stack 로그 수집 및 분석 파이프라인"
description: >
  #daily morning study
category: 
    - dms
    - dms-devops
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## ELK/EFK Stack이란

ELK Stack은 로그 수집, 저장, 시각화를 위한 오픈소스 도구 조합이다.

- **E**lasticsearch: 분산 검색 및 분석 엔진 (저장 + 검색)
- **L**ogstash: 로그 수집 및 변환 파이프라인
- **K**ibana: Elasticsearch 데이터 시각화 대시보드

Kubernetes 환경에서는 Logstash 대신 **Fluentd**나 **Fluent Bit**을 사용하는 경우가 많아서 **EFK Stack**이라고 부른다. Fluentd는 경량이고 Kubernetes와 통합이 잘 되어 있다.

---

## 왜 로그 집중화가 필요한가

마이크로서비스 환경에서는 수십~수백 개의 컨테이너가 동시에 동작한다. 각 서비스 로그가 개별 파드에 흩어져 있으면:

- 특정 요청의 전체 흐름을 추적하기 어렵다
- 파드가 재시작되면 로그가 사라진다
- 장애 발생 시 원인 파악에 시간이 많이 걸린다

로그 집중화(Centralized Logging)로 이런 문제를 해결한다.

---

## ELK Stack 아키텍처

```
[애플리케이션/서버]
       ↓
  [Logstash / Beats]   ← 수집 에이전트
       ↓
  [Elasticsearch]      ← 저장 및 인덱싱
       ↓
   [Kibana]            ← 시각화 및 검색
```

### 각 구성 요소 역할

**Beats (Filebeat, Metricbeat 등)**

로그를 수집해서 Elasticsearch나 Logstash로 전송하는 경량 에이전트다. Logstash보다 훨씬 가볍고 파드나 호스트에 사이드카/DaemonSet으로 배포한다.

```yaml
# Kubernetes에서 Filebeat를 DaemonSet으로 배포
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: filebeat
spec:
  selector:
    matchLabels:
      app: filebeat
  template:
    spec:
      containers:
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:8.12.0
        volumeMounts:
        - name: varlog
          mountPath: /var/log
          readOnly: true
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

**Logstash**

수집한 로그를 파싱, 필터링, 변환해서 Elasticsearch로 전송한다. 데이터 파이프라인 처리 기능이 강력하다.

```
# Logstash 파이프라인 설정 예시
input {
  beats {
    port => 5044
  }
}

filter {
  grok {
    match => {
      "message" => "%{COMBINEDAPACHELOG}"
    }
  }
  date {
    match => ["timestamp", "dd/MMM/yyyy:HH:mm:ss Z"]
  }
  mutate {
    remove_field => ["host"]
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "weblog-%{+YYYY.MM.dd}"
  }
}
```

**Elasticsearch**

로그 데이터를 인덱싱하고 빠른 검색을 제공한다. 분산 시스템으로 수평 확장이 가능하고, 역색인(Inverted Index) 구조로 텍스트 검색이 매우 빠르다.

```bash
# Elasticsearch REST API로 로그 검색
GET /weblog-2026.09.01/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "status": "500" } },
        {
          "range": {
            "@timestamp": {
              "gte": "now-1h",
              "lte": "now"
            }
          }
        }
      ]
    }
  }
}
```

---

## EFK Stack (Kubernetes 환경)

Kubernetes에서는 EFK(Elasticsearch + Fluentd + Kibana) 조합을 많이 쓴다.

```
[K8s Pod 로그]
      ↓
 [Fluent Bit]     ← 각 노드에 DaemonSet으로 배포, 로그 수집
      ↓
  [Fluentd]       ← 집계/변환 (선택 사항)
      ↓
[Elasticsearch]
      ↓
  [Kibana]
```

**Fluent Bit vs Fluentd**

| 항목 | Fluent Bit | Fluentd |
|------|-----------|---------|
| 메모리 사용 | ~1MB | ~40MB |
| 성능 | 매우 빠름 | 빠름 |
| 플러그인 | 제한적 | 풍부함 |
| 용도 | 엣지/컨테이너 로그 수집 | 복잡한 변환 파이프라인 |

일반적으로 Fluent Bit으로 노드에서 로그를 수집하고, Fluentd로 집계/가공한 뒤 Elasticsearch에 저장하는 2단계 구조를 사용한다.

---

## Elasticsearch 핵심 개념

### 인덱스(Index)

데이터베이스의 테이블과 비슷한 개념이다. 로그 데이터는 날짜별로 인덱스를 분리하는 것이 일반적이다.

```
weblog-2026.09.01
weblog-2026.09.02
...
```

### 샤드(Shard)와 복제본(Replica)

- **샤드**: 인덱스를 분산 저장하는 단위. 여러 노드에 나눠 저장해 성능과 용량을 확장한다.
- **레플리카 샤드**: 샤드의 복제본. 노드 장애 시 데이터를 보호하고 읽기 성능을 높인다.

```json
// 인덱스 생성 시 샤드와 레플리카 설정
PUT /weblog
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1
  }
}
```

### ILM (Index Lifecycle Management)

로그 인덱스는 시간이 지나면 쌓여서 스토리지를 과도하게 사용한다. ILM으로 인덱스 생명주기를 자동 관리한다.

```
Hot Phase (최근 로그, 빠른 스토리지)
   ↓ 7일 후
Warm Phase (자주 조회 안함, 느린 스토리지)
   ↓ 30일 후
Cold Phase (거의 조회 안함)
   ↓ 90일 후
Delete Phase (삭제)
```

---

## Kibana 활용

### Discover

인덱스에서 로그를 실시간으로 검색하고 필터링한다. Lucene 쿼리 문법을 사용한다.

```
# 500 에러 중 /api/login 경로 로그 검색
status:500 AND url:"/api/login"

# 특정 사용자의 최근 1시간 활동
user_id:"user123" AND @timestamp:[now-1h TO now]
```

### Dashboard

여러 시각화(파이 차트, 라인 그래프, 데이터 테이블 등)를 모아 대시보드를 만든다.

- HTTP 상태 코드 분포 (파이 차트)
- 시간대별 요청 수 (라인 그래프)
- 응답 시간 히트맵
- 에러율 트렌드

### Alerting

특정 조건이 충족되면 Slack, PagerDuty, 이메일 등으로 알림을 보낸다.

```
조건 예시:
- 5분 내 500 에러가 50건 이상 발생하면 Slack 알림
- 평균 응답 시간이 2초를 초과하면 PagerDuty 알림
```

---

## OpenSearch (AWS 환경)

AWS에서는 Elasticsearch의 오픈소스 포크인 **OpenSearch**를 사용하는 경우가 많다. Amazon OpenSearch Service로 완전 관리형으로 사용할 수 있다.

```
[CloudWatch Logs / Kinesis Firehose]
              ↓
    [Amazon OpenSearch Service]
              ↓
    [OpenSearch Dashboards (Kibana 대체)]
```

---

## 실무에서 자주 쓰는 패턴

### 구조화된 로그 (Structured Logging)

JSON 형식으로 로그를 출력하면 파싱 과정이 단순해지고 필드 검색이 쉬워진다.

```json
// 구조화된 로그 예시
{
  "timestamp": "2026-09-01T09:00:00Z",
  "level": "ERROR",
  "service": "order-service",
  "trace_id": "abc123",
  "user_id": "user456",
  "message": "Payment processing failed",
  "error_code": "PAYMENT_TIMEOUT",
  "duration_ms": 5023
}
```

### Correlation ID (요청 추적)

마이크로서비스 환경에서 요청이 여러 서비스를 거칠 때, 동일한 `trace_id`를 헤더로 전파하고 모든 로그에 포함시키면 Kibana에서 단일 요청의 전체 흐름을 추적할 수 있다.

### 로그 레벨 전략

```
DEBUG  → 개발 환경에서만 수집
INFO   → 주요 비즈니스 이벤트 (API 요청/응답, 배치 완료 등)
WARN   → 잠재적 문제 (재시도, 낮은 재고 경고 등)
ERROR  → 즉각 대응 필요한 오류 (결제 실패, DB 연결 오류 등)
```

프로덕션에서 DEBUG 레벨까지 모두 수집하면 스토리지와 비용이 급증한다. 보통 INFO 이상만 Elasticsearch로 보내고, 필요할 때만 DEBUG를 활성화한다.

---

## 스토리지 비용 관리

로그 데이터는 빠르게 쌓인다. 실무에서 비용을 줄이는 방법들:

1. **ILM 정책 설정**: 오래된 인덱스를 자동으로 삭제하거나 콜드 스토리지로 이동
2. **샘플링**: 정상 요청 로그는 10%만 수집하고 에러 로그는 100% 수집
3. **압축**: Elasticsearch의 `best_compression` 코덱 사용
4. **필드 최소화**: 불필요한 필드는 Logstash/Fluentd 단계에서 제거

---

## 요약

| 구성요소 | 역할 |
|---------|------|
| Filebeat / Fluent Bit | 각 서버/컨테이너에서 로그 수집 |
| Logstash / Fluentd | 로그 파싱, 필터링, 변환 |
| Elasticsearch | 인덱싱 및 검색 |
| Kibana / OpenSearch Dashboards | 시각화 및 대시보드 |

ELK/EFK Stack은 구축과 운영 비용이 있지만, 분산 환경에서 로그 집중화와 빠른 장애 대응을 위한 핵심 인프라다. 최근에는 Grafana Loki처럼 더 가벼운 대안도 있으므로 규모와 요구사항에 맞게 선택하는 것이 중요하다.
