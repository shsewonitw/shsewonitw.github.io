---
layout: post
title: "[Daily morning study] 분산 락(Distributed Lock) 개념과 구현 방법"
description: >
  #daily morning study
category: 
    - dms
    - dms-backend
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 분산 락이란?

단일 서버 환경에서는 `synchronized` 키워드나 `Mutex`, `Semaphore`로 공유 자원 접근을 제어할 수 있다. 하지만 여러 서버 인스턴스가 동시에 동작하는 분산 환경에서는 프로세스 레벨 락이 의미 없다. 각 서버가 독립된 메모리 공간을 갖기 때문이다.

**분산 락(Distributed Lock)**은 분산 환경에서 여러 노드가 동시에 같은 자원에 접근하는 것을 막기 위한 상호 배제(Mutual Exclusion) 메커니즘이다.

### 분산 락이 필요한 상황

- 재고 차감: 동시에 여러 서버가 같은 상품의 재고를 차감할 때 중복 차감 방지
- 중복 결제 방지: 동일한 주문에 대한 결제 요청이 여러 인스턴스에 도달할 때
- 스케줄러: 여러 인스턴스 중 단 하나만 배치 작업을 실행해야 할 때
- 캐시 갱신: 여러 노드가 동시에 캐시 미스 후 DB 조회 → Thundering Herd 문제 방지

---

## 분산 락 구현 방법 비교

| 방법 | 구현체 | 특징 | 단점 |
|------|--------|------|------|
| Redis SETNX | Redisson, Lettuce | 빠르다, 단순하다 | Redis 단일 장애 시 문제 |
| Redlock | Redis 클러스터 | 고가용성 | 복잡도, 시간 동기화 의존 |
| ZooKeeper | Curator | 강한 일관성 | 느림, 운영 복잡도 높음 |
| DB 기반 | 직접 구현 | 별도 인프라 불필요 | DB 부하, 성능 한계 |

---

## Redis SETNX 기반 락

### 기본 원리

`SETNX`(SET if Not eXists)는 키가 없을 때만 값을 설정하는 원자적 명령어다. 이 원자성 덕분에 여러 클라이언트가 동시에 시도해도 하나만 성공한다.

```bash
# 락 획득 시도
SET lock_key unique_value NX PX 30000
# NX: 키가 없을 때만 설정
# PX 30000: 30초 후 자동 만료 (TTL)
```

핵심 포인트:
- **NX 옵션**: 원자적 잠금 획득 보장
- **TTL 설정**: 락 보유 클라이언트가 죽어도 자동으로 해제됨 (데드락 방지)
- **unique_value**: 자신이 설정한 락만 해제하기 위한 식별자

### 락 해제 (Lua 스크립트)

락 해제 시 "자신의 락인지 확인" + "삭제"를 원자적으로 수행해야 한다. 단순히 `DEL`을 쓰면 TTL 만료로 이미 다른 클라이언트가 획득한 락을 실수로 해제할 수 있다.

```lua
-- Lua 스크립트로 원자적 해제
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

```python
import redis
import uuid

client = redis.Redis()

def acquire_lock(lock_name, expire_ms=30000):
    token = str(uuid.uuid4())
    result = client.set(lock_name, token, nx=True, px=expire_ms)
    if result:
        return token
    return None

def release_lock(lock_name, token):
    script = """
    if redis.call("GET", KEYS[1]) == ARGV[1] then
        return redis.call("DEL", KEYS[1])
    else
        return 0
    end
    """
    client.eval(script, 1, lock_name, token)

# 사용 예시
token = acquire_lock("inventory:product_1")
if token:
    try:
        # 재고 차감 로직
        pass
    finally:
        release_lock("inventory:product_1", token)
```

---

## Redisson 사용 (Java)

Java 생태계에서는 Redisson 라이브러리가 분산 락을 편리하게 제공한다.

```java
RLock lock = redissonClient.getLock("inventory:product_1");

try {
    // 최대 10초 대기, 획득 후 30초 만료
    boolean acquired = lock.tryLock(10, 30, TimeUnit.SECONDS);
    if (acquired) {
        // 임계 구역
        deductInventory(productId);
    }
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
} finally {
    if (lock.isHeldByCurrentThread()) {
        lock.unlock();
    }
}
```

Redisson의 장점:
- 워치독(Watchdog) 기능: 락 보유 중 TTL을 자동으로 연장해서 처리 중 락 만료 방지
- tryLock으로 대기 시간 제어
- Pub/Sub 기반 락 대기: 스핀락 대신 Redis의 키 만료 알림을 이용해 CPU 절약

---

## Redlock 알고리즘

단일 Redis 노드 기반 락은 Redis 장애 시 락 보장이 깨진다. Redlock은 이를 해결하기 위해 **N개(보통 5개)의 독립된 Redis 인스턴스**를 사용한다.

### 과정

1. 현재 시간을 밀리초 단위로 기록 (`t1`)
2. 동일한 키/값으로 **N개 인스턴스 모두에** 순차적으로 락 획득 시도 (각각 짧은 타임아웃)
3. 과반수 이상(`N/2 + 1`)의 인스턴스에서 획득 성공 && 경과 시간이 TTL보다 짧으면 → 락 유효
4. 실패 시 모든 인스턴스에 해제 요청

```
총 5개 Redis 인스턴스
→ 3개 이상에서 획득 성공 + 획득 소요 시간 < TTL
→ 유효 락 획득
```

### Redlock의 논란

Martin Kleppmann이 Redlock의 안전성 문제를 지적했다:
- 클라이언트가 GC Pause(Stop-the-World) 중 TTL이 만료되면 락 보장 깨짐
- NTP 시간 조정으로 시간이 앞/뒤로 뛰면 만료 계산 오류 발생

결론: **강한 일관성이 필요한 금융 거래**에는 ZooKeeper나 etcd 기반 락이 더 적합하다.

---

## ZooKeeper 기반 락

ZooKeeper는 Paxos 합의 알고리즘 기반으로 강한 일관성을 보장한다.

### 임시 순서 노드를 이용한 락

```
/locks/
  ├── lock-0000000001  ← 락 보유 중 (가장 작은 번호)
  ├── lock-0000000002  ← 대기 중
  └── lock-0000000003  ← 대기 중
```

1. 클라이언트가 `/locks/lock-` 접두사로 임시 순서 노드를 생성
2. 생성된 노드 번호가 가장 작으면 락 획득
3. 아니라면 바로 앞 번호 노드에 Watch 등록
4. 앞 노드가 삭제되면(락 해제 or 클라이언트 장애) Watch 이벤트로 깨어나 재시도

클라이언트 장애 시 임시 노드가 자동 삭제되므로 데드락이 발생하지 않는다.

```java
// Apache Curator 사용 예시
InterProcessMutex lock = new InterProcessMutex(client, "/locks/inventory");

try {
    if (lock.acquire(10, TimeUnit.SECONDS)) {
        deductInventory(productId);
    }
} finally {
    lock.release();
}
```

---

## DB 기반 락

별도 인프라 없이 기존 DB로 구현하는 방식이다.

### 낙관적 락 (Optimistic Lock)

충돌이 드물다고 가정. `version` 컬럼을 이용해 수정 시 충돌을 감지한다.

```sql
-- 조회 시 버전 함께 가져옴
SELECT id, stock, version FROM inventory WHERE product_id = 1;

-- 업데이트 시 버전 조건 추가
UPDATE inventory
SET stock = stock - 1, version = version + 1
WHERE product_id = 1 AND version = :현재버전;

-- 영향받은 행이 0이면 충돌 → 재시도
```

JPA에서는 `@Version` 어노테이션으로 간편하게 사용할 수 있다.

### 비관적 락 (Pessimistic Lock)

충돌이 잦다고 가정. `SELECT FOR UPDATE`로 트랜잭션 동안 행을 잠근다.

```sql
BEGIN;
SELECT stock FROM inventory WHERE product_id = 1 FOR UPDATE;
-- 다른 트랜잭션은 이 행에 접근 불가
UPDATE inventory SET stock = stock - 1 WHERE product_id = 1;
COMMIT;
```

DB 락의 한계:
- 트랜잭션 범위 밖(애플리케이션 레벨 처리)에는 사용 불가
- 고트래픽에서 DB 커넥션 고갈 위험
- 분산 DB(샤딩 환경)에서는 적용 어려움

---

## 실무 선택 기준

| 상황 | 추천 방식 |
|------|----------|
| 단순 상호 배제, 빠른 응답 필요 | Redis SETNX (Redisson) |
| Redis 단일 장애 허용 불가 | Redlock 또는 ZooKeeper |
| 강한 일관성 필요 (금융, 결제) | ZooKeeper / etcd |
| 기존 DB만 사용 가능 | 낙관적 락 (충돌 적음) / 비관적 락 (충돌 잦음) |
| 배치 단일 실행 보장 | ShedLock (Spring) — DB/Redis 모두 지원 |

---

## 분산 락 사용 시 주의사항

**1. TTL은 충분히 길게, 하지만 데드락 방지 가능한 수준으로**
- 처리 시간의 2~3배 정도가 일반적
- Redisson Watchdog처럼 자동 갱신 기능 활용

**2. 락 재진입(Reentrancy) 고려**
- 같은 스레드/프로세스가 락을 중복 획득해야 하는 경우 처리 필요
- Redisson RLock은 재진입 락 지원

**3. 락 범위를 최소화**
- 락 보유 시간 동안 외부 API 호출이나 긴 I/O 작업은 피할 것
- 락 내부 코드는 빠르게 처리하고 즉시 해제

**4. 페일오버 시나리오 테스트**
- Redis/ZooKeeper 장애 시 서비스가 어떻게 동작하는지 미리 확인
- Circuit Breaker 패턴과 함께 사용하면 장애 전파 차단 가능
