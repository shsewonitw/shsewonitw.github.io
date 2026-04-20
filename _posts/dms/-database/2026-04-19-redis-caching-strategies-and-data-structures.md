---
layout: post
title: "[Daily morning study] Redis 캐싱 전략과 핵심 자료구조"
description: >
  #daily morning study
category: 
    - dms
    - dms-database
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## Redis란?

Redis(Remote Dictionary Server)는 인메모리 기반의 키-값 저장소다. 데이터를 RAM에 저장하기 때문에 디스크 기반 DB보다 월등히 빠르다(보통 읽기 100,000 ops/sec 이상). 캐시, 세션 저장소, 메시지 브로커, 리더보드 등 다양한 용도로 사용된다.

---

## 핵심 자료구조

Redis는 단순한 문자열 저장소가 아니라 다양한 자료구조를 지원한다.

| 자료구조 | 명령어 예시 | 주요 사용 사례 |
|---|---|---|
| String | `SET`, `GET`, `INCR` | 캐시, 카운터, 세션 |
| List | `LPUSH`, `RPOP`, `LRANGE` | 큐, 최근 방문 기록 |
| Set | `SADD`, `SMEMBERS`, `SINTER` | 태그, 좋아요 목록, 집합 연산 |
| Sorted Set | `ZADD`, `ZRANGE`, `ZRANK` | 랭킹, 우선순위 큐 |
| Hash | `HSET`, `HGET`, `HGETALL` | 객체 저장, 유저 프로필 |
| Bitmap | `SETBIT`, `BITCOUNT` | 출석 체크, feature flag |
| HyperLogLog | `PFADD`, `PFCOUNT` | 대략적인 유니크 카운트 |

### String

가장 기본적인 자료구조. 문자열뿐만 아니라 정수도 저장하고 원자적으로 증가/감소시킬 수 있다.

```bash
SET user:1:name "Alice"
GET user:1:name          # "Alice"

SET visit_count 0
INCR visit_count         # 1 (원자적 증가)
INCRBY visit_count 5     # 6
```

TTL(Time-To-Live)을 설정해서 자동 만료되게 할 수 있다.

```bash
SET session:abc123 "user_data" EX 3600   # 1시간 후 자동 삭제
TTL session:abc123                        # 남은 시간(초) 반환
```

### Hash

관련된 필드를 하나의 키 아래에 묶어서 저장한다. 유저 객체처럼 여러 속성을 가진 데이터에 적합하다.

```bash
HSET user:1 name "Alice" age 30 email "alice@example.com"
HGET user:1 name          # "Alice"
HGETALL user:1            # 모든 필드-값 쌍 반환
HINCRBY user:1 age 1      # age를 1 증가
```

String으로 JSON 직렬화해서 저장하는 것보다 Hash가 나은 경우: 일부 필드만 읽거나 수정할 때 전체를 역직렬화할 필요 없이 해당 필드만 접근 가능하다.

### Sorted Set

각 멤버에 score를 부여하고, score 기준으로 정렬된 집합이다. 랭킹 시스템 구현에 최적화되어 있다.

```bash
ZADD leaderboard 3500 "Alice"
ZADD leaderboard 4200 "Bob"
ZADD leaderboard 2800 "Charlie"

ZRANGE leaderboard 0 -1 WITHSCORES    # score 오름차순
ZREVRANGE leaderboard 0 2 WITHSCORES  # 상위 3명 (내림차순)
ZRANK leaderboard "Alice"             # 0-based 순위
ZSCORE leaderboard "Bob"              # 4200
```

### List

양방향 연결 리스트 구조로, 양 끝에서 O(1)로 삽입/삭제 가능하다. 큐(LPUSH + RPOP)나 스택(LPUSH + LPOP)으로 활용한다.

```bash
LPUSH job_queue "job3" "job2" "job1"  # 왼쪽에 삽입
RPOP job_queue                         # 오른쪽에서 꺼냄 (FIFO)
LRANGE job_queue 0 -1                  # 전체 조회
```

---

## 캐싱 전략

### Cache Aside (Lazy Loading)

가장 일반적인 패턴. 캐시를 먼저 조회하고, 없으면 DB에서 읽어서 캐시에 저장한다.

```
Read:
  1. 캐시 조회
  2. Cache Hit → 캐시 데이터 반환
  3. Cache Miss → DB 조회 → 캐시 저장 → 데이터 반환

Write:
  1. DB에 쓰기
  2. 캐시 무효화(invalidate) 또는 업데이트
```

```python
def get_user(user_id):
    cache_key = f"user:{user_id}"
    
    # 캐시 조회
    cached = redis.get(cache_key)
    if cached:
        return json.loads(cached)
    
    # Cache Miss → DB 조회
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    
    # 캐시 저장 (TTL 10분)
    redis.setex(cache_key, 600, json.dumps(user))
    return user
```

**장점**: 실제로 필요한 데이터만 캐시에 올라감. 캐시 장애 시에도 DB로 폴백 가능.  
**단점**: 첫 요청은 항상 느림(Cache Miss). DB와 캐시 간 일관성 관리 필요.

### Write Through

DB에 쓸 때 동시에 캐시에도 쓴다.

```
Write:
  1. DB에 쓰기
  2. 캐시에도 동일하게 쓰기
```

**장점**: 캐시가 항상 최신 데이터 유지.  
**단점**: 쓰기 지연 증가. 한 번도 읽히지 않는 데이터도 캐시에 올라가 메모리 낭비.

### Write Behind (Write Back)

캐시에만 먼저 쓰고, 비동기로 DB에 반영한다.

**장점**: 쓰기 성능 극대화.  
**단점**: 캐시 장애 시 데이터 유실 위험. 구현 복잡도 높음.

---

## Cache Invalidation 전략

캐싱에서 가장 어려운 문제 중 하나가 데이터 일관성 유지다.

### TTL(Time-To-Live) 설정

가장 단순한 방법. 일정 시간 후 자동으로 캐시가 만료된다.

```bash
SET key value EX 300    # 5분 후 만료
```

TTL을 너무 짧게 설정하면 캐시 효과가 줄고, 너무 길게 설정하면 stale 데이터 제공 위험이 생긴다.

### 이벤트 기반 무효화

데이터 변경 시 관련 캐시 키를 명시적으로 삭제한다.

```python
def update_user(user_id, data):
    db.update("UPDATE users SET ... WHERE id = ?", user_id, data)
    redis.delete(f"user:{user_id}")   # 캐시 무효화
```

### Cache Stampede (캐시 스탬피드) 문제

인기 있는 캐시 키가 만료되는 순간 다수의 요청이 동시에 DB로 몰리는 현상이다.

**해결책 1: Mutex Lock**

```python
def get_popular_data(key):
    cached = redis.get(key)
    if cached:
        return cached
    
    lock_key = f"lock:{key}"
    if redis.set(lock_key, 1, nx=True, ex=5):  # 락 획득
        try:
            data = db.query(...)
            redis.setex(key, 300, data)
            return data
        finally:
            redis.delete(lock_key)
    else:
        # 락 획득 실패 → 잠시 대기 후 캐시 재조회
        time.sleep(0.1)
        return redis.get(key)
```

**해결책 2: Probabilistic Early Expiration**  
캐시가 만료되기 직전에 확률적으로 미리 갱신한다. 만료 임박 시 일부 요청이 선제적으로 캐시를 갱신하여 stampede를 방지한다.

---

## 메모리 관리: Eviction Policy

Redis 메모리가 가득 차면 Eviction Policy에 따라 키를 삭제한다.

| 정책 | 설명 |
|---|---|
| `noeviction` | 삭제 안 함, 에러 반환 (기본값) |
| `allkeys-lru` | 전체 키 중 LRU(가장 오래 사용 안 한) 삭제 |
| `volatile-lru` | TTL 있는 키 중 LRU 삭제 |
| `allkeys-lfu` | 전체 키 중 LFU(가장 적게 사용한) 삭제 |
| `volatile-ttl` | TTL 짧은 키 우선 삭제 |
| `allkeys-random` | 전체 키 중 랜덤 삭제 |

캐시 서버로 사용할 때는 보통 `allkeys-lru` 또는 `allkeys-lfu`를 사용한다.

```bash
# redis.conf 또는 명령어로 설정
CONFIG SET maxmemory 256mb
CONFIG SET maxmemory-policy allkeys-lru
```

---

## 데이터 영속성

Redis는 인메모리 저장소지만 두 가지 방식으로 디스크에 저장할 수 있다.

**RDB (Redis Database Backup)**  
특정 시점의 스냅샷을 덤프 파일로 저장. 저장 주기 사이의 데이터는 손실 가능.

**AOF (Append Only File)**  
모든 쓰기 명령을 로그로 기록. 데이터 손실 최소화. 파일 크기가 커지는 단점.

캐시 전용으로 사용할 때는 영속성을 비활성화해 성능을 높이는 경우도 많다.

---

## 실무에서 자주 쓰는 패턴

### 세션 저장소

```python
# 로그인 시
session_id = generate_session_id()
redis.setex(f"session:{session_id}", 86400, json.dumps(user_info))

# 요청마다 검증
session = redis.get(f"session:{session_id}")
```

### 분산 락(Distributed Lock)

여러 서버에서 동시에 실행되면 안 되는 작업(ex: 재고 차감)에 사용한다.

```python
lock_acquired = redis.set("lock:inventory", 1, nx=True, ex=30)
if lock_acquired:
    # 임계 영역 실행
    ...
    redis.delete("lock:inventory")
```

### Rate Limiting

API 요청 수를 제한할 때 INCR + TTL을 활용한다.

```python
def is_rate_limited(user_id, limit=100):
    key = f"rate:{user_id}:{int(time.time() // 60)}"  # 분 단위 버킷
    count = redis.incr(key)
    if count == 1:
        redis.expire(key, 60)  # 최초 설정 시 TTL 부여
    return count > limit
```
