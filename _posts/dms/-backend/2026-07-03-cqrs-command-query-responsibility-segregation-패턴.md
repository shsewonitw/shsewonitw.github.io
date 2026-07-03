---
layout: post
title: "[Daily morning study] CQRS(Command Query Responsibility Segregation) 패턴"
description: >
  #daily morning study
category: 
    - dms
    - dms-backend
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## CQRS란?

CQRS(Command Query Responsibility Segregation)는 **명령(Command)** 과 **조회(Query)** 의 책임을 분리하는 아키텍처 패턴이다.

전통적인 CRUD 모델에서는 데이터를 읽고 쓰는 모델이 하나다. CQRS는 이를 둘로 나눈다.

- **Command**: 데이터를 변경하는 작업 (생성, 수정, 삭제). 반환값 없음(또는 ID만 반환).
- **Query**: 데이터를 읽는 작업. 사이드 이펙트 없음.

이 아이디어는 Bertrand Meyer의 CQS(Command Query Separation) 원칙에서 출발했고, Greg Young이 이를 아키텍처 수준으로 발전시켜 CQRS라고 명명했다.

---

## 왜 분리하는가?

### 읽기와 쓰기의 특성이 다르다

| 특성 | Command (쓰기) | Query (읽기) |
|------|--------------|-------------|
| 빈도 | 상대적으로 낮음 | 상대적으로 높음 |
| 복잡도 | 비즈니스 규칙, 유효성 검사 | 복잡한 조인, 집계 |
| 확장 방식 | 트랜잭션 처리 최적화 | 읽기 성능 최적화 |
| 모델 형태 | 도메인 중심 | 화면/보고서 중심 |

읽기 트래픽이 쓰기보다 10배, 100배 많은 경우가 흔하다. 같은 모델을 공유하면 각각에 맞는 최적화가 어렵다.

---

## 기본 구조

```
Client
  │
  ├─ Command → Command Handler → Write Model (DB)
  │
  └─ Query  → Query Handler  → Read Model (DB or View)
```

### Command 쪽

- Command 객체: 의도를 담은 DTO (`CreateOrderCommand`, `CancelOrderCommand`)
- Command Handler: 도메인 모델을 통해 비즈니스 로직 처리 후 Write DB에 반영
- 도메인 이벤트를 발행해서 Read Model을 업데이트하는 경우도 많음

### Query 쪽

- Query 객체: 필요한 데이터와 필터 조건 (`GetOrdersByUserQuery`)
- Query Handler: Read DB에서 직접 DTO를 조회해 반환
- ORM의 복잡한 도메인 모델 없이 raw SQL이나 간단한 쿼리로 처리 가능

---

## 코드 예시 (Node.js / TypeScript)

### Command

```typescript
// command
class CreateOrderCommand {
  constructor(
    public readonly userId: string,
    public readonly items: OrderItem[],
  ) {}
}

// handler
class CreateOrderHandler {
  async execute(command: CreateOrderCommand): Promise<string> {
    const order = Order.create(command.userId, command.items);
    await this.orderRepository.save(order);
    await this.eventBus.publish(new OrderCreatedEvent(order.id));
    return order.id;
  }
}
```

### Query

```typescript
// query
class GetUserOrdersQuery {
  constructor(public readonly userId: string) {}
}

// handler — Read DB에서 바로 flat DTO 조회
class GetUserOrdersHandler {
  async execute(query: GetUserOrdersQuery): Promise<OrderSummaryDto[]> {
    return this.db.query(
      `SELECT id, status, total_price, created_at
       FROM orders_view
       WHERE user_id = $1
       ORDER BY created_at DESC`,
      [query.userId],
    );
  }
}
```

Query 핸들러는 도메인 모델(`Order` 클래스)을 전혀 거치지 않는다. 화면에 필요한 형태 그대로 가져온다.

---

## Read Model 동기화 방법

Write DB와 Read DB를 분리할 때 두 모델을 어떻게 동기화하느냐가 핵심 과제다.

### 1. 동기 업데이트

Command가 처리된 직후 같은 트랜잭션 또는 직후 호출로 Read Model을 업데이트.

- 장점: 구현이 단순, 일관성 보장
- 단점: Command 처리 성능에 영향

### 2. 이벤트 기반 비동기 업데이트

도메인 이벤트를 발행하고, 이벤트 핸들러가 Read Model을 별도로 갱신.

```
Order 저장
  └─ OrderCreatedEvent 발행
       └─ (비동기) OrderReadModelUpdater → Read DB 갱신
```

- 장점: Command 성능에 영향 없음, Read/Write DB를 완전히 다른 저장소(예: MySQL ↔ Elasticsearch)로 분리 가능
- 단점: **최종 일관성(Eventual Consistency)** — 이벤트 처리 전까지 Read Model이 오래된 데이터를 반환할 수 있음

### 3. Event Sourcing과 결합

Write Model은 현재 상태 대신 **이벤트 로그**만 저장. Read Model은 이벤트를 재생(replay)해서 뷰를 만든다.

CQRS와 Event Sourcing은 독립적인 패턴이지만 함께 자주 사용된다.

---

## 언제 도입하면 좋은가

CQRS가 적합한 경우:
- 읽기/쓰기 트래픽 불균형이 심한 서비스
- 조회 요구사항이 복잡하고 다양한 경우 (대시보드, 검색, 리포트)
- MSA에서 서비스별로 읽기 전용 모델이 필요한 경우
- Event Sourcing 도입을 고려 중인 경우

CQRS가 과한 경우:
- 단순 CRUD 애플리케이션
- 팀 규모가 작고 복잡도를 감당하기 어려운 경우
- 강한 일관성이 반드시 필요한 경우 (최종 일관성을 허용하기 어려울 때)

---

## 장단점 정리

### 장점

- **읽기 성능 최적화**: Read Model을 조회에 특화된 형태로 설계 가능
- **도메인 모델 단순화**: Command 쪽 도메인 모델이 조회 요건에 오염되지 않음
- **독립적 확장**: Read/Write를 별도 인프라로 스케일 아웃 가능
- **명확한 의도 표현**: Command 이름이 비즈니스 언어를 직접 표현 (`PlaceOrder`, `CancelReservation`)

### 단점

- **복잡도 증가**: 클래스, 핸들러, 이벤트 수가 늘어남
- **최종 일관성**: 비동기 동기화 시 Read Model 지연 발생 가능
- **운영 복잡도**: Write/Read DB가 분리되면 관리 포인트 증가
- **학습 비용**: 팀 전체가 패턴을 이해해야 효과적으로 동작

---

## 요약

CQRS는 읽기와 쓰기의 관심사를 분리함으로써 각각에 맞는 최적화를 가능하게 하는 패턴이다. 단순한 애플리케이션에는 오버엔지니어링이지만, 복잡한 도메인 로직과 다양한 조회 요건이 충돌하는 시스템에서는 유지보수성과 성능 모두를 개선할 수 있다.
