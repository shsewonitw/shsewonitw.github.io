---
layout: post
title: "[Daily morning study] Event Sourcing (이벤트 소싱) 개념과 CQRS와의 관계"
description: >
  #daily morning study
category: 
    - dms
    - dms-backend
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## Event Sourcing이란

일반적인 시스템은 현재 상태(state)만 데이터베이스에 저장한다. 계좌 잔액을 업데이트하면 이전 잔액은 사라진다.

Event Sourcing은 이와 반대로 **상태 변화(state change)를 이벤트 시퀀스로 저장**한다. 현재 상태는 이벤트를 처음부터 순서대로 재생(replay)해서 구한다.

```
일반 방식: users 테이블에 balance = 50000 저장

Event Sourcing:
  [AccountOpened: balance=0]
  [MoneyDeposited: amount=100000]
  [MoneyWithdrawn: amount=30000]
  [MoneyDeposited: amount=80000]
  → 현재 잔액 = 0 + 100000 - 30000 + 80000 = 150000
```

이벤트는 과거에 일어난 사실(fact)이므로 **절대 수정하거나 삭제하지 않는다**. 이벤트 스토어(event store)에 append-only 방식으로 추가만 한다.

---

## 핵심 개념

### 이벤트 (Event)

이벤트는 도메인에서 발생한 사실을 나타낸다. 명명 규칙은 과거형 동사를 사용한다.

```json
{
  "eventId": "uuid-1234",
  "aggregateId": "account-9999",
  "eventType": "MoneyDeposited",
  "occurredAt": "2026-07-22T09:00:00Z",
  "payload": {
    "amount": 50000,
    "currency": "KRW"
  }
}
```

### 애그리게이트 (Aggregate)

연관된 이벤트의 묶음. 각 애그리게이트는 고유한 ID를 가지며, 해당 ID에 대한 모든 이벤트를 재생해서 현재 상태를 계산한다.

```python
class BankAccount:
    def __init__(self):
        self.balance = 0
        self.is_closed = False

    def apply(self, event):
        if event.type == "AccountOpened":
            self.balance = 0
        elif event.type == "MoneyDeposited":
            self.balance += event.payload["amount"]
        elif event.type == "MoneyWithdrawn":
            self.balance -= event.payload["amount"]
        elif event.type == "AccountClosed":
            self.is_closed = True

    @classmethod
    def load(cls, events):
        account = cls()
        for event in events:
            account.apply(event)
        return account
```

### 스냅샷 (Snapshot)

이벤트가 수천 개 쌓이면 매번 전체를 재생하는 게 느려진다. 이를 해결하기 위해 **특정 시점의 상태를 스냅샷으로 저장**한다.

```
스냅샷 없음: 이벤트 1 ~ 10000 전체 재생
스냅샷 있음: 이벤트 9000 시점 스냅샷 로드 → 이벤트 9001 ~ 10000만 재생
```

스냅샷은 보통 N번째 이벤트마다, 또는 상태 복원 시간이 임계치를 넘을 때 생성한다.

---

## Event Sourcing과 CQRS의 관계

Event Sourcing은 CQRS(Command Query Responsibility Segregation)와 자연스럽게 결합된다.

| 역할 | 처리 방식 |
| --- | --- |
| Command (쓰기) | 이벤트를 생성해서 이벤트 스토어에 저장 |
| Query (읽기) | 이벤트를 구독해서 읽기 전용 뷰(Read Model) 업데이트 |

```
[Client]
    │
    ▼ Command (입금 요청)
[Command Handler]
    │ 애그리게이트 이벤트 생성
    ▼
[Event Store] ──publish──▶ [Event Bus]
                                │
                         [Projection]
                                │
                         [Read Model DB]
                                │
                         ◀── Query (잔액 조회)
```

이벤트 스토어가 쓰기의 원천(source of truth)이 되고, 읽기용 데이터는 이벤트를 기반으로 여러 형태로 파생된다. 같은 이벤트를 여러 Projection이 구독해서 서로 다른 형태의 뷰를 만들 수 있다.

---

## 장점

**완전한 감사 로그(Audit Log)**
모든 변경 이력이 이벤트로 남으므로 "언제, 무엇이, 왜 바뀌었는지" 추적이 가능하다. 금융, 의료처럼 이력 추적이 중요한 도메인에 특히 유용하다.

**시간 여행(Time Travel)**
특정 시점의 상태를 이벤트 재생으로 복원할 수 있다. 버그 재현이나 특정 시점 데이터 분석에 활용된다.

**이벤트 기반 통합**
이벤트가 생성될 때 다른 서비스나 시스템에 발행(publish)하면 쉽게 이벤트 드리븐 아키텍처를 구성할 수 있다.

**디버깅 용이성**
프로덕션에서 발생한 문제를 이벤트 시퀀스를 그대로 재생해서 로컬에서 재현할 수 있다.

---

## 단점

**쿼리 복잡도 증가**
현재 상태를 직접 조회할 수 없어 Read Model을 따로 유지해야 한다. 시스템이 복잡해진다.

**최종 일관성(Eventual Consistency)**
이벤트 발행 → Projection 업데이트 사이에 지연이 있다. 쓰기 직후 읽기 시 오래된 데이터를 반환할 수 있다.

**이벤트 스키마 변경 어려움**
저장된 이벤트는 수정할 수 없으므로 이벤트 구조 변경 시 버전 관리 전략이 필요하다. upcaster 패턴을 사용해서 구버전 이벤트를 최신 버전으로 변환한다.

**학습 곡선**
전통적인 CRUD 방식과 사고방식이 달라서 팀 전체가 패러다임에 익숙해지는 데 시간이 걸린다.

---

## 이벤트 스키마 버전 관리

```python
def upcast_event(event):
    if event["eventType"] == "MoneyDeposited":
        if "version" not in event:
            # v1 → v2: currency 필드 추가
            event["payload"]["currency"] = "KRW"
            event["version"] = 2
    return event
```

이벤트 저장 시 version 필드를 포함하고, 로드 시 최신 버전으로 변환하는 upcaster를 체이닝한다.

---

## 적합한 사용 시나리오

Event Sourcing이 적합한 경우:
- 변경 이력이 중요한 도메인 (금융 거래, 의료 기록, 주문 관리)
- 복잡한 비즈니스 규칙을 가진 도메인
- 이벤트 드리븐 마이크로서비스 아키텍처
- 감사(audit) 요구사항이 엄격한 시스템

Event Sourcing이 부적합한 경우:
- 단순 CRUD 위주의 시스템
- 이벤트보다 현재 상태 쿼리가 훨씬 많은 경우
- 작은 팀이 빠르게 개발해야 하는 경우

---

## 이벤트 스토어 구현 예시

```sql
CREATE TABLE events (
    id          BIGSERIAL PRIMARY KEY,
    aggregate_id UUID NOT NULL,
    event_type  VARCHAR(100) NOT NULL,
    version     INT NOT NULL,
    payload     JSONB NOT NULL,
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (aggregate_id, version)
);

-- 특정 애그리게이트의 이벤트 조회
SELECT * FROM events
WHERE aggregate_id = 'account-9999'
ORDER BY version ASC;
```

`(aggregate_id, version)` 유니크 제약으로 동시 쓰기 충돌을 방지한다 (낙관적 잠금과 동일한 효과).

---

## 정리

| 항목 | 일반 상태 저장 | Event Sourcing |
| --- | --- | --- |
| 저장 대상 | 현재 상태 | 이벤트 시퀀스 |
| 쓰기 방식 | UPDATE / DELETE | Append-Only |
| 이력 추적 | 별도 감사 테이블 필요 | 기본 제공 |
| 읽기 복잡도 | 낮음 | 높음 (Read Model 필요) |
| 일관성 | 강한 일관성 | 최종 일관성 |

Event Sourcing은 강력하지만 복잡성 비용이 크다. 도메인의 감사/이력 요구사항과 팀 역량을 먼저 평가하고 도입 여부를 결정하는 게 좋다.
