---
layout: post
title: "[Daily morning study] 서킷 브레이커 패턴 (Circuit Breaker Pattern)"
description: >
  #daily morning study
category: 
    - dms
    - dms-backend
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 서킷 브레이커 패턴이란

마이크로서비스 환경에서는 서비스 A가 서비스 B를 호출하고, B가 C를 호출하는 식으로 연쇄 의존이 생긴다. 이때 C가 응답 불능 상태가 되면 B의 스레드가 블로킹되고, 그게 A까지 퍼지면서 전체 시스템이 마비된다. 이를 **Cascading Failure(연쇄 장애)** 라고 한다.

서킷 브레이커(Circuit Breaker)는 전기 회로의 차단기처럼 작동한다. 특정 서비스 호출이 연속으로 실패하면 회로를 "열어(Open)" 더 이상 요청을 보내지 않고 즉시 에러를 반환한다. 이를 통해 장애 전파를 차단하고 시스템 전체의 안정성을 확보한다.

---

## 3가지 상태

서킷 브레이커는 세 가지 상태를 오간다.

| 상태 | 설명 |
|------|------|
| **Closed** | 정상 상태. 요청이 그대로 통과한다. 실패율을 집계하다가 임계치를 넘으면 Open으로 전환. |
| **Open** | 차단 상태. 모든 요청을 즉시 거부(Fail Fast)한다. 일정 시간 후 Half-Open으로 전환. |
| **Half-Open** | 탐색 상태. 제한된 수의 요청만 통과시켜 서비스가 복구됐는지 확인한다. 성공하면 Closed, 실패하면 다시 Open으로 돌아간다. |

상태 전환 흐름:

```
Closed ──(실패율 > 임계치)──▶ Open
  ▲                              │
  │                        (타임아웃 경과)
  │                              ▼
  └──(일정 성공 확인)──── Half-Open
```

---

## Fail Fast의 의미

Open 상태에서 즉시 에러를 반환하는 것을 Fail Fast라고 한다. 겉으로는 실패처럼 보이지만 실제로는 방어 전략이다.

- 느린 실패 대신 빠른 실패로 호출 스레드가 블로킹되지 않는다
- 이미 장애 중인 서비스에 불필요한 트래픽을 보내지 않아 복구를 방해하지 않는다
- 클라이언트 입장에서도 수 초를 기다리다 타임아웃되는 것보다 즉각 에러를 받는 게 더 낫다

---

## Fallback 처리

서킷이 열렸을 때 단순히 에러를 반환하는 것보다, 미리 정의한 대체 응답을 돌려주는 게 더 좋은 UX를 제공한다. 이를 Fallback이라고 한다.

```java
// Resilience4j 예시 (Java)
CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("inventoryService");

Supplier<String> decoratedSupplier = CircuitBreaker.decorateSupplier(
    circuitBreaker,
    () -> inventoryService.getStock(productId)
);

String result = Try.ofSupplier(decoratedSupplier)
    .recover(throwable -> "재고 정보를 일시적으로 불러올 수 없습니다.")
    .get();
```

Fallback 전략 예시:

- 캐시된 이전 응답 반환
- 기본값(default value) 반환
- 다른 서비스(보조 서버)로 우회
- 사용자에게 "현재 서비스 점검 중" 메시지 표시

---

## 주요 설정값

서킷 브레이커의 동작은 보통 다음 파라미터로 조정한다.

| 파라미터 | 설명 | 예시 |
|----------|------|------|
| `failureRateThreshold` | Closed → Open 전환 기준 실패율(%) | `50` (50% 이상 실패 시 Open) |
| `slidingWindowSize` | 실패율 계산에 사용할 최근 호출 수 | `10` (최근 10번 기준) |
| `waitDurationInOpenState` | Open 상태 유지 시간 | `10s` |
| `permittedCallsInHalfOpen` | Half-Open 상태에서 허용할 테스트 요청 수 | `3` |
| `slowCallRateThreshold` | 느린 응답 비율 임계치(%) | `80` |
| `slowCallDurationThreshold` | 느린 응답 기준 시간 | `2s` |

실패율뿐 아니라 **느린 응답(Slow Call)** 도 서킷을 열 수 있다. 응답이 오긴 하지만 2초씩 걸린다면 그것도 시스템에 부하가 된다.

---

## 슬라이딩 윈도우 방식

실패율을 계산하는 방식이 두 가지다.

**Count-based(횟수 기반)**: 최근 N번의 호출 결과로 실패율 계산.

```
[성공, 실패, 실패, 성공, 실패, 실패, 실패, 성공, 실패, 실패]
 → 7/10 = 70% 실패 → 임계치(50%) 초과 → Open
```

**Time-based(시간 기반)**: 최근 N초 동안의 호출 결과로 계산.

```
최근 60초 동안 100번 호출 중 60번 실패
 → 60% 실패 → 임계치 초과 → Open
```

Count-based는 트래픽이 적을 때 민감하게 반응하고, Time-based는 트래픽 변동에 더 안정적이다.

---

## Resilience4j 실제 설정 예시

Spring Boot + Resilience4j 기준 application.yml 설정:

```yaml
resilience4j:
  circuitbreaker:
    instances:
      inventoryService:
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
        permittedNumberOfCallsInHalfOpenState: 3
        slowCallDurationThreshold: 2s
        slowCallRateThreshold: 80
        registerHealthIndicator: true
```

서비스에 적용:

```java
@Service
public class OrderService {

    @CircuitBreaker(name = "inventoryService", fallbackMethod = "getStockFallback")
    public int getStock(String productId) {
        return inventoryClient.getStock(productId);
    }

    private int getStockFallback(String productId, Exception e) {
        return 0; // 재고 0으로 처리 (기본값)
    }
}
```

---

## 서킷 브레이커 vs 재시도(Retry)

두 패턴은 함께 쓰이지만 목적이 다르다.

| | 서킷 브레이커 | 재시도 |
|-|---------------|--------|
| **목적** | 장애 전파 차단 | 일시적 오류 극복 |
| **동작** | 실패 시 즉시 차단 | 실패 시 N번 재시도 |
| **적합한 상황** | 서비스 자체가 다운됐을 때 | 네트워크 순간 패킷 손실 등 |

재시도를 무한정 하면 다운된 서비스에 더 많은 부하를 준다. 그래서 보통 **재시도(Retry) → 서킷 브레이커(Circuit Breaker)** 순으로 감싸서, 재시도가 모두 실패하면 서킷이 열리도록 구성한다.

---

## 모니터링

서킷 브레이커의 상태는 반드시 모니터링해야 한다. Open 상태가 됐다는 것 자체가 장애 신호다.

Resilience4j는 Micrometer와 연동해 Prometheus/Grafana로 다음 메트릭을 노출한다:

- `resilience4j_circuitbreaker_state` — 현재 상태 (0: Closed, 1: Open, 2: Half-Open)
- `resilience4j_circuitbreaker_calls_total` — 호출 수 (성공/실패/무시)
- `resilience4j_circuitbreaker_failure_rate` — 현재 실패율

서킷이 Open 상태로 전환될 때 알림을 받으면 빠른 대응이 가능하다.

---

## 정리

- 서킷 브레이커는 연쇄 장애를 막기 위한 Resilience 패턴이다
- Closed → Open → Half-Open 3가지 상태를 전환하며 동작한다
- Open 상태에서 Fail Fast로 호출 스레드를 보호하고, Fallback으로 대체 응답을 제공한다
- 실패율뿐 아니라 느린 응답도 서킷을 열 수 있다
- Retry 패턴과 함께 사용하고, 반드시 상태 모니터링을 구성해야 한다
