---
layout: post
title: "[Daily morning study] Transactional Outbox 패턴 — 분산 시스템의 원자적 메시지 발행"
description: >
  #daily morning study
category: 
    - dms
    - -backend
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 문제 정의: 이중 쓰기(Dual Write) 문제

마이크로서비스에서 가장 흔하게 마주치는 문제 중 하나가 **이중 쓰기**다. 예를 들어 주문 서비스가 주문을 DB에 저장하면서 동시에 결제 서비스나 이벤트 버스(Kafka 등)에 메시지를 보내야 하는 상황을 생각해보자.

```text
OrderService:
  1. DB에 주문 저장 (commit)
  2. Kafka에 OrderCreated 이벤트 발행

문제:
  - 1번 성공, 2번 실패 → 주문은 저장됐지만 이벤트 유실
  - 2번 성공, 1번 실패 → 이벤트는 나갔지만 DB엔 주문 없음
```

DB 트랜잭션과 메시지 발행은 서로 다른 리소스다. 분산 트랜잭션(2PC) 없이 이 두 작업을 원자적으로 처리하는 것은 불가능하다. 그래서 등장한 패턴이 **Transactional Outbox Pattern**이다.

---

## Outbox 패턴의 핵심 아이디어

핵심은 **메시지를 DB 트랜잭션 안에서 같은 DB에 먼저 기록**하고, 별도의 프로세스가 이 테이블을 읽어 실제 메시지 브로커로 전달하는 것이다.

```text
[ 애플리케이션 트랜잭션 ]
  - orders 테이블에 주문 INSERT
  - outbox 테이블에 OrderCreated 이벤트 INSERT
  → 하나의 DB 트랜잭션으로 commit (원자성 보장)

[ Message Relay 프로세스 ]
  - outbox 테이블을 폴링 or CDC로 감지
  - Kafka / RabbitMQ 등 브로커에 이벤트 발행
  - 발행 완료 후 outbox 레코드를 삭제 or 상태 업데이트
```

이 방식을 사용하면 DB가 단일 진실의 원천(Single Source of Truth)이 되어 이중 쓰기 문제를 해결할 수 있다.

---

## Outbox 테이블 구조

```sql
CREATE TABLE outbox (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type VARCHAR(100) NOT NULL,  -- 'Order', 'Payment' 등
    aggregate_id   VARCHAR(100) NOT NULL,  -- 도메인 객체 ID
    event_type     VARCHAR(100) NOT NULL,  -- 'OrderCreated', 'OrderCancelled' 등
    payload        JSONB        NOT NULL,  -- 이벤트 데이터
    created_at     TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    processed_at   TIMESTAMPTZ,           -- 발행 완료 시각
    status         VARCHAR(20)  NOT NULL DEFAULT 'PENDING'
                                          -- PENDING | SENT | FAILED
);
```

애플리케이션 코드 예시 (Spring + JPA):

```java
@Transactional
public Order createOrder(CreateOrderRequest req) {
    // 1. 도메인 객체 저장
    Order order = orderRepository.save(Order.from(req));

    // 2. 동일 트랜잭션에서 outbox에 이벤트 기록
    OutboxEvent event = OutboxEvent.builder()
        .aggregateType("Order")
        .aggregateId(order.getId().toString())
        .eventType("OrderCreated")
        .payload(objectMapper.valueToTree(new OrderCreatedPayload(order)))
        .build();
    outboxRepository.save(event);

    return order;
}
// 두 INSERT가 하나의 트랜잭션 → 원자성 보장
```

---

## Message Relay 구현 방식

### 방식 1: Polling Publisher

주기적으로 outbox 테이블을 SELECT해서 미처리 레코드를 브로커에 발행하는 가장 단순한 방법이다.

```java
@Scheduled(fixedDelay = 1000)
public void relay() {
    List<OutboxEvent> pending = outboxRepository
        .findByStatusOrderByCreatedAtAsc("PENDING", PageRequest.of(0, 100));

    for (OutboxEvent event : pending) {
        try {
            kafkaTemplate.send(event.getEventType(), event.getPayload().toString());
            event.markSent();
        } catch (Exception e) {
            event.markFailed();
        }
        outboxRepository.save(event);
    }
}
```

**장점**: 구현이 간단하고 별도 인프라가 필요 없다.

**단점**: 폴링 주기만큼 지연이 생기고, DB에 지속적인 폴링 부하가 발생한다.

### 방식 2: CDC (Change Data Capture)

Debezium 같은 CDC 도구를 사용해 DB의 WAL(Write-Ahead Log)을 실시간으로 읽어 outbox 테이블의 변경사항을 감지한다.

```text
PostgreSQL WAL → Debezium Connector → Kafka (outbox.events 토픽)
```

Debezium은 **Outbox Event Router**라는 SMT(Single Message Transform)를 제공한다. 이를 통해 outbox 테이블의 레코드를 자동으로 적절한 Kafka 토픽으로 라우팅할 수 있다.

```json
// Debezium Outbox Event Router 설정 예시
{
  "transforms": "outbox",
  "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
  "transforms.outbox.table.field.event.id": "id",
  "transforms.outbox.table.field.event.key": "aggregate_id",
  "transforms.outbox.table.field.event.type": "event_type",
  "transforms.outbox.table.field.event.payload": "payload",
  "transforms.outbox.route.by.field": "aggregate_type"
}
```

**장점**: 거의 실시간 전달, DB 폴링 부하 없음, 발행 확인 후 삭제 처리가 더 깔끔하다.

**단점**: Debezium, Kafka Connect 등 추가 인프라가 필요하고 운영 복잡도가 높아진다.

---

## At-Least-Once 전달과 멱등성

Outbox 패턴은 메시지를 **최소 한 번(at-least-once)** 전달한다. Relay가 메시지를 브로커로 보낸 후 outbox를 업데이트하기 전에 크래시가 나면 같은 메시지가 재전송될 수 있다.

따라서 소비자 쪽에서 **멱등한 처리**가 필요하다.

```java
// 소비자 예시: 처리한 이벤트 ID를 별도 테이블에 저장
@KafkaListener(topics = "OrderCreated")
@Transactional
public void handle(OrderCreatedEvent event) {
    if (processedEvents.exists(event.getId())) {
        return; // 중복 이벤트 무시
    }
    paymentService.process(event);
    processedEvents.save(event.getId());
}
```

---

## Outbox 패턴 vs 다른 접근법 비교

| 방식 | 원자성 | 복잡도 | 인프라 요구사항 |
|------|--------|--------|----------------|
| 이중 쓰기 (Dual Write) | ❌ | 낮음 | 없음 |
| 2PC (분산 트랜잭션) | ✅ | 매우 높음 | XA 트랜잭션 지원 |
| Outbox (Polling) | ✅ | 보통 | 없음 |
| Outbox (CDC) | ✅ | 높음 | Debezium, Kafka Connect |
| Saga 패턴 | 최종 일관성 | 높음 | 메시지 브로커 |

---

## 실제 적용 시 고려사항

**outbox 테이블 크기 관리**: 처리 완료된 레코드를 주기적으로 아카이브하거나 삭제해야 한다. 누적되면 폴링 성능이 저하된다.

**인덱스 설계**:
```sql
-- 폴링 방식에서 미처리 레코드를 빠르게 조회
CREATE INDEX idx_outbox_status_created ON outbox(status, created_at)
WHERE status = 'PENDING';
```

**재시도 정책**: FAILED 상태의 레코드를 몇 번까지, 얼마나 자주 재시도할지 명확히 정의해야 한다. 영구 실패(dead letter)를 위한 별도 처리도 필요하다.

**페이로드 직렬화**: payload를 JSONB로 저장하면 유연하지만, 스키마 변경 시 역직렬화 오류가 발생할 수 있다. Avro나 Protobuf로 버전 관리하는 것도 고려할 만하다.

---

## 정리

Outbox 패턴은 분산 시스템에서 **DB 쓰기와 메시지 발행의 원자성**을 보장하는 실용적인 해법이다. 2PC의 복잡성과 성능 오버헤드 없이도 데이터 일관성을 유지할 수 있다. Polling 방식은 간단히 시작할 수 있고, 실시간성과 확장성이 중요해지면 CDC로 전환하는 전략이 일반적이다. 단, at-least-once 특성 때문에 소비자 쪽의 멱등성 보장은 항상 세트로 설계해야 한다.
