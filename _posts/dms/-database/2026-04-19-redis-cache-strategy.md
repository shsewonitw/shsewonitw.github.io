---
layout: post
title: "[Daily morning study] Redis 캐시 전략 (Cache-Aside, Write-Through, Write-Behind)"
description: >
  #daily morning study
category: 
    - dms
    - dms-database
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 왜 캐시가 필요한가

DB에 쿼리를 날릴 때마다 디스크 I/O가 발생한다. 동일한 데이터를 반복해서 읽는 패턴이 있다면, 그 결과를 메모리에 저장해두고 바로 돌려주는 게 훨씬 빠르다. Redis는 인메모리 데이터 스토어로, 평균 응답 시간이 1ms 미만이라 이런 캐시 레이어로 많이 쓰인다.

캐시를 도입할 때 핵심 결정 사항은 두 가지다.

1. **읽기 전략**: 캐시에 데이터가 없을 때(Cache Miss) 어떻게 채울 것인가
2. **쓰기 전략**: 데이터가 변경될 때 캐시와 DB를 어떻게 동기화할 것인가

---

## 읽기 전략

### Cache-Aside (Lazy Loading)

가장 흔하게 쓰이는 패턴이다. 애플리케이션이 캐시를 직접 관리한다.

```
1. 앱 → Redis 조회
2. Cache Hit → 바로 반환
3. Cache Miss → DB 조회 → 결과를 Redis에 저장 → 반환
```

```python
def get_user(user_id: str):
    cached = redis.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)

    user = db.query("SELECT * FROM users WHERE id = %s", user_id)
    redis.setex(f"user:{user_id}", 3600, json.dumps(user))  # TTL 1시간
    return user
```

**장점**
- 실제로 읽히는 데이터만 캐싱되므로 메모리 낭비가 적다
- Redis가 다운되어도 DB로 폴백할 수 있어 장애 내성이 높다

**단점**
- 처음 요청(Cache Miss)은 항상 느리다 (Cold Start)
- DB 데이터가 바뀌어도 TTL이 만료되기 전까지 캐시는 오래된 데이터를 반환한다 (Stale Data 문제)

### Read-Through

Cache-Aside와 비슷하지만 캐시 라이브러리나 별도 레이어가 DB 조회까지 담당한다. 앱 코드는 항상 캐시에만 요청한다.

```
앱 → 캐시 레이어 → (Miss면) DB 조회 → 캐시 저장 → 반환
```

앱 코드가 단순해지지만, 캐시 레이어가 DB 스키마를 알아야 하므로 결합도가 높아진다.

---

## 쓰기 전략

### Write-Through

데이터를 쓸 때 캐시와 DB에 **동시에** 저장한다.

```
앱 → Redis 저장 → DB 저장 → 완료 응답
```

```python
def update_user(user_id: str, data: dict):
    # 캐시 먼저 업데이트
    redis.setex(f"user:{user_id}", 3600, json.dumps(data))
    # DB도 바로 업데이트
    db.execute("UPDATE users SET ... WHERE id = %s", user_id)
```

**장점**
- 캐시와 DB가 항상 동기화되어 Stale Data 문제가 없다
- Cache-Aside의 첫 Miss 문제가 없다

**단점**
- 쓰기가 두 번 발생하므로 쓰기 레이턴시가 늘어난다
- 실제로 다시 읽히지 않는 데이터도 캐시에 쌓인다 (불필요한 캐시 오염)

### Write-Behind (Write-Back)

캐시에만 먼저 쓰고, DB 반영은 **비동기**로 나중에 처리한다.

```
앱 → Redis 저장 → 완료 응답
               ↓ (비동기)
             DB 저장
```

**장점**
- 쓰기 응답이 매우 빠르다
- DB에 쓰기가 몰리는 상황(Write-Heavy)에서 부하를 분산할 수 있다

**단점**
- Redis가 DB에 반영하기 전에 다운되면 데이터가 유실된다
- 구현 복잡도가 높다 (비동기 큐, 실패 재시도 로직 필요)
- 금융 트랜잭션처럼 데이터 내구성이 중요한 시스템에는 부적합하다

### Write-Around

쓰기 시 캐시를 건드리지 않고 DB에만 저장한다. 읽을 때 Cache Miss가 나면 그때 캐시를 채운다.

일회성 데이터나 자주 읽히지 않는 데이터에 적합하다. 대용량 로그 저장, 배치 처리 결과 등이 예시다.

---

## 전략 비교

| 전략 | 읽기 성능 | 쓰기 성능 | 일관성 | 장애 내성 |
|------|-----------|-----------|--------|-----------|
| Cache-Aside | Cache Hit 시 빠름 | 보통 | 낮음 (Stale 가능) | 높음 |
| Read-Through | Cache Hit 시 빠름 | 보통 | 낮음 (Stale 가능) | 중간 |
| Write-Through | 빠름 | 느림 | 높음 | 중간 |
| Write-Behind | 빠름 | 매우 빠름 | 중간 | 낮음 |
| Write-Around | 첫 읽기 느림 | 보통 | 보통 | 높음 |

---

## 캐시 무효화 (Cache Invalidation)

캐시 전략에서 가장 어려운 부분이 무효화다. 유명한 말이 있다.

> "There are only two hard things in Computer Science: cache invalidation and naming things."

### TTL (Time-To-Live)

```python
redis.setex("key", 3600, value)  # 1시간 후 자동 삭제
```

단순하지만 TTL 만료 전까지는 Stale Data를 서빙하게 된다.

### 이벤트 기반 무효화

DB 데이터가 변경될 때 해당 캐시 키를 직접 삭제하거나 업데이트한다.

```python
def update_user(user_id: str, data: dict):
    db.execute("UPDATE users SET ... WHERE id = %s", user_id)
    redis.delete(f"user:{user_id}")  # 캐시 삭제 → 다음 읽기 때 DB에서 채워짐
```

삭제 후 다음 읽기가 Cache-Aside로 채워지는 구조다. 업데이트와 삭제 사이에 짧은 시간 동안 Stale Data가 서빙될 수 있다.

### 캐시 스탬피드 (Cache Stampede) 문제

TTL이 만료되는 순간 동시에 수백 개의 요청이 Cache Miss를 받으면, 모두 DB를 동시에 조회한다. DB에 갑자기 부하가 몰리는 현상이다.

**해결 방법**

1. **뮤텍스 락**: 첫 번째 요청만 DB를 조회하게 하고, 나머지는 대기

```python
def get_with_lock(key: str):
    cached = redis.get(key)
    if cached:
        return cached

    lock_key = f"lock:{key}"
    if redis.set(lock_key, "1", nx=True, ex=5):  # 락 획득
        try:
            value = db.query(...)
            redis.setex(key, 3600, value)
            return value
        finally:
            redis.delete(lock_key)
    else:
        # 락 획득 실패 → 잠시 대기 후 재시도
        time.sleep(0.1)
        return get_with_lock(key)
```

2. **확률적 조기 갱신 (Probabilistic Early Expiration)**: TTL이 완전히 만료되기 전에 일부 요청에서 미리 갱신

---

## Redis 데이터 구조 선택

캐싱 시나리오에 따라 Redis 자료구조를 적절히 선택하면 성능을 더 끌어올릴 수 있다.

| 시나리오 | Redis 자료구조 | 명령어 예시 |
|----------|---------------|-------------|
| 단순 키-값 캐싱 | String | `GET`, `SETEX` |
| 유저 세션, 객체 캐싱 | Hash | `HGETALL`, `HSET` |
| 최근 조회 목록 | List | `LPUSH`, `LRANGE` |
| 랭킹, 정렬된 목록 | Sorted Set | `ZADD`, `ZRANGE` |
| 중복 제거 (방문자 수 등) | Set / HyperLogLog | `SADD`, `PFADD` |

세션 캐싱에 Hash를 쓰면 필드 단위로 업데이트가 가능해서 String보다 효율적이다.

```python
# String: 전체 객체를 직렬화/역직렬화
redis.set("session:abc", json.dumps({"user_id": 1, "role": "admin"}))

# Hash: 필드만 업데이트 가능
redis.hset("session:abc", "role", "user")  # 다른 필드는 그대로
```

---

## 캐시 히트율 모니터링

캐시가 제대로 동작하는지 히트율(Hit Rate)을 지속적으로 확인해야 한다.

```bash
# Redis 통계 확인
redis-cli INFO stats | grep -E "keyspace_hits|keyspace_misses"

# keyspace_hits: 캐시 히트 횟수
# keyspace_misses: 캐시 미스 횟수
# Hit Rate = hits / (hits + misses)
```

일반적으로 히트율이 80% 미만이면 캐싱 전략을 재검토할 필요가 있다.

---

## 정리

- **읽기 위주 서비스**: Cache-Aside + TTL 기반 무효화로 시작해서 히트율 확인
- **쓰기 빈도가 낮고 일관성이 중요**: Write-Through
- **쓰기가 많고 속도가 우선**: Write-Behind (단, 데이터 유실 위험 감수)
- **캐시 스탬피드 방지**: 뮤텍스 락 또는 확률적 조기 갱신
- **모니터링**: 히트율 추적 필수, 70~80% 미만이면 전략 재검토
