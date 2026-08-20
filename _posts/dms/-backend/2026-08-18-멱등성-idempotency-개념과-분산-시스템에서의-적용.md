---
layout: post
title: "[Daily morning study] 멱등성(Idempotency) 개념과 분산 시스템에서의 적용"
description: >
  #daily morning study
category: 
    - dms
    - -backend
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 멱등성(Idempotency)이란

멱등성이란 동일한 요청을 한 번 보내든 여러 번 보내든 결과가 같은 성질을 말한다. 수학에서 온 개념으로, `f(f(x)) = f(x)`가 성립하는 연산을 의미한다.

분산 시스템, 특히 REST API나 메시지 큐 기반 아키텍처에서 네트워크 장애나 타임아웃으로 인해 **같은 요청이 두 번 이상 처리될 수 있는 상황**이 빈번히 발생한다. 이 때 멱등성을 보장하면 중복 처리로 인한 데이터 불일치나 부작용을 방지할 수 있다.

---

## HTTP 메서드별 멱등성

| HTTP 메서드 | 멱등성 | 안전성(Safe) | 설명 |
|------------|--------|------------|------|
| GET        | O      | O          | 조회만 하므로 항상 안전하고 멱등 |
| HEAD       | O      | O          | GET과 동일하나 본문 없음 |
| PUT        | O      | X          | 동일 데이터로 여러 번 갱신해도 결과 동일 |
| DELETE     | O      | X          | 두 번 삭제해도 첫 번째 이후 상태 동일 |
| POST       | X      | X          | 호출마다 새 리소스 생성 — 기본적으로 비멱등 |
| PATCH      | X (조건부) | X     | 연산 방식에 따라 달라짐 |

PUT은 `"나이를 30으로 설정"` 방식이라 여러 번 호출해도 동일하지만, PATCH는 `"나이를 1 증가"` 방식이면 호출마다 결과가 달라지므로 비멱등이 된다.

---

## 왜 POST가 문제가 되는가

결제, 주문 생성 같은 POST 요청에서 클라이언트가 타임아웃을 받고 재시도를 하면 서버는 동일 요청을 두 번 처리할 수 있다.

```
클라이언트 → [결제 요청] → 서버
클라이언트 ← [타임아웃]  ← 서버 (처리는 됐지만 응답 유실)
클라이언트 → [결제 재시도] → 서버  ← 중복 결제 발생!
```

이 문제를 해결하는 가장 일반적인 방법이 **Idempotency Key(멱등성 키)** 패턴이다.

---

## Idempotency Key 패턴

클라이언트가 요청 시 고유 키를 함께 전송하고, 서버는 이 키를 기반으로 중복 요청을 감지해 이미 처리된 결과를 반환한다.

```http
POST /payments HTTP/1.1
Content-Type: application/json
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000

{
  "amount": 10000,
  "currency": "KRW",
  "target_account": "1234-5678"
}
```

### 서버 처리 흐름

```
1. Idempotency-Key를 DB/캐시에서 조회
2. 이미 존재하면 → 저장된 이전 응답 그대로 반환
3. 존재하지 않으면 → 요청 처리 후 결과를 키와 함께 저장
```

### 구현 예시 (Node.js + Redis)

```javascript
async function handlePayment(idempotencyKey, paymentData) {
  const cached = await redis.get(`idempotency:${idempotencyKey}`);
  
  if (cached) {
    // 이미 처리된 요청 — 저장된 결과 반환
    return JSON.parse(cached);
  }

  // 처리 중 상태를 먼저 기록 (다른 인스턴스의 동시 처리 방지)
  await redis.set(
    `idempotency:${idempotencyKey}`,
    JSON.stringify({ status: 'processing' }),
    'EX', 86400  // 24시간 TTL
  );

  try {
    const result = await processPayment(paymentData);
    // 최종 결과로 업데이트
    await redis.set(
      `idempotency:${idempotencyKey}`,
      JSON.stringify(result),
      'EX', 86400
    );
    return result;
  } catch (err) {
    // 실패 시 키 삭제하여 재시도 허용
    await redis.del(`idempotency:${idempotencyKey}`);
    throw err;
  }
}
```

실제 Stripe, PayPal 같은 결제 시스템이 이 방식을 공식 지원한다.

---

## 데이터베이스 레벨의 멱등성

### UPSERT 활용

```sql
-- PostgreSQL: INSERT 충돌 시 UPDATE로 처리
INSERT INTO orders (order_id, user_id, amount, status)
VALUES ('order-uuid-001', 42, 10000, 'pending')
ON CONFLICT (order_id) DO UPDATE
  SET status = EXCLUDED.status,
      updated_at = NOW();
```

order_id에 UNIQUE 제약이 있어서 같은 주문을 두 번 삽입해도 두 번째는 충돌 처리되므로 멱등하다.

### 조건부 업데이트로 멱등성 보장

```sql
-- 버전 기반 낙관적 락
UPDATE accounts
SET balance = balance - 10000, version = version + 1
WHERE account_id = 'acc-001'
  AND version = 5;  -- 기대하는 버전이 맞을 때만 처리
```

`version`이 이미 변경됐다면 영향받은 행이 0이 되어 재처리를 감지할 수 있다.

---

## 메시지 큐에서의 멱등성

Kafka, RabbitMQ 등에서 메시지는 at-least-once 전달 방식이기 때문에 컨슈머가 같은 메시지를 두 번 받을 수 있다. 이를 처리하는 방법:

### 1. 처리 이력 테이블 관리

```sql
CREATE TABLE processed_messages (
  message_id VARCHAR(255) PRIMARY KEY,
  processed_at TIMESTAMP DEFAULT NOW()
);
```

```python
def consume_message(message):
    message_id = message['id']
    
    # 이미 처리했는지 확인
    if db.exists('processed_messages', message_id):
        return  # 중복 — 건너뜀
    
    # 비즈니스 로직 처리
    process_order(message['data'])
    
    # 처리 이력 기록
    db.insert('processed_messages', {'message_id': message_id})
```

이 패턴은 메시지 처리와 이력 기록을 같은 트랜잭션 안에 묶어야 완전한 보장이 된다.

### 2. Kafka의 Exactly-Once Semantics

Kafka 0.11부터 트랜잭셔널 API를 통해 exactly-once를 지원한다.

```java
producer.initTransactions();

try {
    producer.beginTransaction();
    producer.send(new ProducerRecord<>("output-topic", key, value));
    producer.sendOffsetsToTransaction(offsets, consumerGroupMetadata);
    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
}
```

---

## 멱등성과 트랜잭션의 관계

멱등성 키 저장과 실제 비즈니스 로직을 동일 트랜잭션에서 처리해야 완전한 보장이 된다.

```
나쁜 예:
1. 비즈니스 로직 처리 (트랜잭션 A)
2. 멱등성 키 저장 (트랜잭션 B) ← 1과 2 사이에 장애 나면 키 없이 처리 완료됨
```

```
좋은 예:
단일 트랜잭션에서:
  1. 멱등성 키 INSERT (중복이면 즉시 반환)
  2. 비즈니스 로직 처리
  3. COMMIT
```

---

## 실제 적용 시 고려 사항

**Idempotency Key 생성**: 클라이언트에서 UUID v4를 생성해 사용하는 게 일반적이다. 서버 측에서는 키를 검증만 하고 생성하지 않는다.

**키의 유효 기간(TTL)**: 멱등성 키를 영구 보관하면 스토리지 낭비가 생긴다. 보통 24시간~7일 사이의 TTL을 설정한다. 클라이언트가 그 이후에 재시도하면 새 요청으로 처리된다.

**요청 본문(Payload)의 변경**: 동일 키에 다른 본문을 보내는 경우를 어떻게 처리할지 정책이 필요하다. Stripe는 키가 동일하면 최초 요청 결과를 항상 반환하고 본문 불일치 시 에러를 내보낸다.

**분산 환경에서의 동시성**: 같은 키로 동시에 두 요청이 오면 Race Condition이 발생할 수 있다. Redis의 `SET NX`(SET if Not Exists) 또는 DB의 UNIQUE 제약을 활용한 락으로 해결한다.

```redis
SET idempotency:key-001 "processing" NX EX 30
# NX: 키가 없을 때만 설정
# 성공하면 처리 진행, 실패하면 이미 다른 인스턴스가 처리 중
```

---

## 정리

멱등성은 분산 시스템의 신뢰성을 높이는 핵심 설계 원칙이다. 네트워크 장애, 재시도, 중복 메시지는 피할 수 없기 때문에 시스템이 그 상황을 자연스럽게 흡수할 수 있어야 한다.

- **GET, PUT, DELETE**: 기본적으로 멱등
- **POST**: Idempotency Key 패턴으로 멱등성 부여
- **메시지 큐**: 처리 이력 테이블 또는 exactly-once semantics 활용
- **DB**: UPSERT, 조건부 업데이트 활용
- **핵심**: 멱등성 키 저장과 비즈니스 로직은 같은 트랜잭션 안에서 처리
