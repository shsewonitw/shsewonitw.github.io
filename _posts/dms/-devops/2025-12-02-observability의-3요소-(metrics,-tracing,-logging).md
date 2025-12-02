---
layout: post
title: "[Daily morning study] Observability의 3요소 (Metrics, Tracing, Logging)"
description: >
  #daily morning study
category: 
    - dms
    - -devops
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---
# Observability의 3요소: Metrics, Tracing, Logging

Observability는 시스템의 상태와 동작을 파악하고, 문제를 진단하기 위한 필수적인 요소다. 이 문서에서는 Observability의 3가지 핵심 요소인 Metrics, Tracing, Logging에 대해 알아보자.

---

## 1. Metrics

Metrics는 시스템의 성능을 수치적으로 표현하는 데이터다. 주로 시간에 따라 변화하는 값들을 측정한다. 다음과 같은 성격을 가진다:

- **주기적 수집**: 특정 시간 간격에 따라 데이터를 수집하고 분석하는 방식이다.
- **집계 및 평균화**: 대량의 데이터를 쉽게 관리하기 위해 집계하거나 평균을 낼 수 있다.
- **모니터링 도구**: Grafana, Prometheus 등의 도구를 활용하여 시각화하고 알림을 받을 수 있다.

### 예시

서버의 CPU 사용률을 측정하는 JSON 형식의 메트릭은 다음과 같다.

```json
{
  "timestamp": "2023-10-01T12:00:00Z",
  "cpu_usage": 75.2,
  "memory_usage": 65.4
}
```

---

## 2. Tracing

Tracing은 각각의 요청이 시스템을 통해 어떻게 흐르는지를 추적하는 기술이다. 주로 분산 시스템에서 사용되며, 다음과 같은 특성을 가진다:

- **요청의 경로**: 요청이 시스템의 어느 부분을 통과했는지에 대한 정보를 제공한다.
- **성능 분석**: 각 서비스 호출에 소요되는 시간을 측정하여 병목 현상을 식별한다.
- **상관 관계**: 다양한 서비스 간의 상관 관계를 분석할 수 있다.

### 예시

OpenTelemetry 라이브러리를 이용하여 Tracing을 구현하는 코드 예시는 다음과 같다.

```python
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

with tracer.start_as_current_span("process_order"):
    # 주문 처리 로직
    with tracer.start_as_current_span("validate_order"):
        # 주문 유효성 검사 로직
    with tracer.start_as_current_span("charge_credit_card"):
        # 카드 요금 청구 로직
```

---

## 3. Logging

Logging은 애플리케이션에서 발생하는 이벤트, 오류 및 상태 정보를 기록하는 행위이다. 주로 문제 해결 및 상태 진단에 사용된다.

- **영구적 저장**: 로그는 파일이나 데이터베이스에 영구적으로 저장되어 나중에 분석할 수 있다.
- **문맥 정보**: 로그에는 오류가 발생한 시간, 요청 데이터, 사용자 정보 등 다양한 문맥 정보를 포함할 수 있다.
- **다양한 로그 수준**: DEBUG, INFO, WARN, ERROR 등 다양한 로그 수준을 설정할 수 있다.

### 예시

Python에서 로그를 기록하는 간단한 코드 예시는 다음과 같다.

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

logger.info("주문이 시작되었습니다.")

try:
    # 주문 처리 로직
    raise Exception("주문 처리 중 오류 발생")
except Exception as e:
    logger.error(f"오류 발생: {e}")
```

---

## 결론

Metrics, Tracing, Logging은 서로 보완적인 관계에 있으며, 각각의 요소를 적절하게 활용하면 시스템의 Observability를 높이고, 장애를 신속하게 진단할 수 있다. 이 3요소 모두가 함께 작동할 때, 개발자는 더 나은 사용자 경험과 시스템 신뢰성을 제공할 수 있다. 

자신의 시스템에서 이 세 가지 요소를 어떻게 구현할지를 고민해보자.
