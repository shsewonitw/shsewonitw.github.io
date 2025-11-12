---
layout: post
title: "[Daily morning study] 트랜잭션(Transaction)의 ACID 속성"
description: >
  #daily morning study
category: 
    - dms
    - -database
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---
# 트랜잭션(Transaction)의 ACID 속성

트랜잭션은 데이터베이스 시스템에서 일관성을 유지하기 위해 필요한 연산의 단위를 의미한다. 복잡한 데이터 처리 중에는 트랜잭션이 반드시 필요하며, 이를 통해 데이터베이스의 무결성이 유지된다. ACID는 트랜잭션이 만족해야 하는 네 가지 속성을 설명하는 약어이며, 다음과 같은 속성들로 구성된다.

## 1. 원자성 (Atomicity)

원자성은 트랜잭션 내의 모든 작업이 완전히 수행되거나 전혀 수행되지 않음을 보장하는 속성이다. 다시 말해, 트랜잭션이 중단되면 그 트랜잭션에서 이루어진 모든 변화는 원래 상태로 롤백된다.

### 예시:
```sql
BEGIN;
INSERT INTO accounts (user_id, balance) VALUES (1, 100);
UPDATE accounts SET balance = balance - 100 WHERE user_id = 2;
COMMIT;
```
만약 두 번째 쿼리에서 오류가 발생하면 첫 번째 쿼리도 무효화되며 데이터는 복구된다.

## 2. 일관성 (Consistency)

일관성은 트랜잭션이 성공적으로 완료되었을 때 데이터베이스가 일관된 상태를 유지해야 함을 의미한다. 트랜잭션이 데이터베이스에 적용되기 전과 후의 상태는 정의된 규칙과 제약 조건을 충족해야 한다.

### 예시:
```sql
CREATE TABLE accounts (
    user_id INT PRIMARY KEY,
    balance DECIMAL CHECK (balance >= 0)
);
```
이 예에서 각 사용자의 `balance`는 0 이상이어야 하며, 트랜잭션이 완료된 후 이 제약이 항상 유지되어야 한다.

## 3. 고립성 (Isolation)

고립성은 동시에 실행되는 트랜잭션이 서로의 작업에 영향을 미치지 않도록 보장하는 속성이다. 트랜잭션이 완료될 때까지 다른 트랜잭션은 그 트랜잭션의 중간 상태를 볼 수 없다.

### 예시:
두 개의 트랜잭션이 같은 데이터에 접근할 때 이를 방지하기 위해 잠금을 사용한다.
```sql
-- 트랜잭션 A
BEGIN;
SELECT * FROM accounts WHERE user_id = 1 FOR UPDATE;

-- 트랜잭션 B
BEGIN;
SELECT * FROM accounts WHERE user_id = 1; -- 이 쿼리는 대기 혹은 실패할 수 있다.
```

## 4. 지속성 (Durability)

지속성은 트랜잭션이 성공적으로 완료되고 나면 그 결과가 영구적으로 데이터베이스에 저장되어야 함을 의미한다. 시스템 장애나 오류가 발생하더라도 완료된 트랜잭션의 데이터는 사라지지 않는다.

### 예시:
```sql
BEGIN;
INSERT INTO accounts (user_id, balance) VALUES (3, 200);
COMMIT;  -- 이 시점에서 변경 사항은 지속적으로 기록된다.
```
트랜잭션이 성공적으로 완료되면 데이터는 디스크에 안전하게 저장된다.

## 요약

트랜잭션의 ACID 속성은 데이터베이스의 안정성과 무결성을 유지하기 위해 필수적이다. 각 속성들은 서로 연결되어 있으며, 이를 통해 데이터베이스 연산이 신뢰할 수 있는 방식으로 이루어질 수 있다. 이러한 원칙을 이해하고 적용하는 것은 데이터베이스 관리의 기본적인 요소이므로, 트랜잭션을 다룰 때 항상 기억해야 한다.

---

편리한 가이드로 ACID 속성을 이해하고 적용해보길 바란다!
