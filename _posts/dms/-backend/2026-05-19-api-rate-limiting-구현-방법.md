---
layout: post
title: "[Daily morning study] API Rate Limiting 구현 방법"
description: >
  #daily morning study
category: 
    - dms
    - dms-backend
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---
# API Rate Limiting 구현 방법

## Rate Limiting이란?

Rate Limiting은 클라이언트가 일정 시간 동안 API를 호출할 수 있는 횟수를 제한하는 기법이다.  
서버 자원을 보호하고, 특정 사용자의 과도한 요청으로 인해 다른 사용자가 서비스를 이용하지 못하는 상황을 방지한다.

## 왜 필요한가

- **DoS / DDoS 방어**: 악의적인 대량 요청으로부터 서버를 보호한다.
- **공정한 자원 분배**: 특정 클라이언트가 API를 독점하는 것을 막는다.
- **비용 제어**: 클라우드 환경에서 무분별한 요청으로 인한 비용 폭증을 방지한다.
- **안정성 보장**: 트래픽 급증 시 서비스 저하를 막는다.

---

## 주요 알고리즘

### 1. Fixed Window Counter

가장 단순한 방식이다. 고정된 시간 창(예: 1분)을 기준으로 요청 수를 카운트한다.

```
|-- 1분 --| |-- 1분 --|
  50 req      50 req
```

**문제점**: 창 경계 근처에서 두 배 요청이 통과될 수 있다.  
예를 들어 00:59에 50개, 01:00에 50개 요청이 오면 2초 안에 100개가 처리된다.

---

### 2. Sliding Window Log

모든 요청의 타임스탬프를 로그로 남기고, 현재 시각 기준으로 최근 N초 안의 요청만 센다.

```python
import time
from collections import deque

class SlidingWindowLog:
    def __init__(self, limit, window_sec):
        self.limit = limit
        self.window = window_sec
        self.log = deque()

    def allow(self):
        now = time.time()
        # 오래된 타임스탬프 제거
        while self.log and self.log[0] < now - self.window:
            self.log.popleft()

        if len(self.log) < self.limit:
            self.log.append(now)
            return True
        return False
```

정확하지만 메모리 사용량이 많다. 요청이 많을수록 로그 크기가 커진다.

---

### 3. Token Bucket

버킷에 토큰이 일정 속도로 채워지고, 요청마다 토큰을 하나씩 소비하는 방식이다.

```
토큰 충전 속도: 10 tokens/sec
버킷 용량: 100 tokens

요청 들어옴 → 토큰 있으면 소비 후 허용
             → 토큰 없으면 거절
```

**특징**
- 버스트 트래픽을 어느 정도 허용한다 (버킷 용량만큼).
- 가장 많이 쓰이는 알고리즘 중 하나다.
- AWS API Gateway, Stripe 등에서 사용한다.

```python
import time

class TokenBucket:
    def __init__(self, capacity, refill_rate):
        self.capacity = capacity
        self.tokens = capacity
        self.refill_rate = refill_rate  # tokens per second
        self.last_refill = time.time()

    def allow(self):
        now = time.time()
        elapsed = now - self.last_refill
        self.tokens = min(self.capacity, self.tokens + elapsed * self.refill_rate)
        self.last_refill = now

        if self.tokens >= 1:
            self.tokens -= 1
            return True
        return False
```

---

### 4. Leaky Bucket

요청을 큐에 넣고 일정한 속도로만 처리하는 방식이다. 네트워크 트래픽 셰이핑에서 유래했다.

```
요청 → [큐] → 일정 속도로 처리
              (초당 10개만 나감)
```

- 처리 속도가 일정하다는 장점이 있지만, 버스트를 완전히 흡수하지 못하고 큐가 넘치면 요청이 버려진다.
- Token Bucket과 개념이 비슷하지만 방향이 반대다.

---

### 5. Sliding Window Counter (실무 권장)

Fixed Window의 경계 문제를 해결하면서도 Sliding Window Log보다 메모리를 적게 쓴다.

현재 창과 이전 창의 카운트를 가중 평균으로 합산한다.

```
현재 창 경과 비율 = 0.3 (30% 지남)
이전 창 가중치   = 1 - 0.3 = 0.7

추정 요청 수 = 이전_카운트 × 0.7 + 현재_카운트
```

Redis의 `INCR` + `EXPIRE` 조합으로 구현하기 쉬워서 실무에서 자주 쓴다.

---

## Redis를 이용한 구현 예시

분산 환경에서는 각 서버가 별도 카운터를 가지면 안 되므로, Redis 같은 중앙 저장소를 사용한다.

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379)

def is_allowed(user_id: str, limit: int, window_sec: int) -> bool:
    key = f"rate_limit:{user_id}:{int(time.time() // window_sec)}"
    
    pipe = r.pipeline()
    pipe.incr(key)
    pipe.expire(key, window_sec * 2)
    results = pipe.execute()
    
    current_count = results[0]
    return current_count <= limit
```

Lua 스크립트를 쓰면 `INCR`과 `EXPIRE`를 원자적으로 처리할 수 있다.

```lua
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])

local count = redis.call('INCR', key)
if count == 1 then
    redis.call('EXPIRE', key, window)
end

if count > limit then
    return 0
else
    return 1
end
```

---

## HTTP 응답 헤더

Rate Limit 정보는 응답 헤더로 클라이언트에 알려주는 것이 관례다.

| 헤더 | 의미 |
|------|------|
| `X-RateLimit-Limit` | 허용된 최대 요청 수 |
| `X-RateLimit-Remaining` | 남은 요청 횟수 |
| `X-RateLimit-Reset` | 카운터 초기화 시각 (Unix timestamp) |
| `Retry-After` | 429 응답 시 재시도까지 대기 시간 (초) |

한도 초과 시 **HTTP 429 Too Many Requests**를 반환한다.

---

## 실무 적용 시 고려사항

**식별 기준 선택**
- IP 기반: 구현이 단순하지만 NAT 환경에서 여러 사용자가 같은 IP를 공유할 수 있다.
- API 키 기반: 인증된 클라이언트를 정확히 구분할 수 있다.
- 사용자 ID 기반: 로그인 사용자에게 적합하다.

**분산 환경 주의사항**
- 여러 인스턴스가 띄워져 있으면 반드시 Redis 같은 공유 저장소를 써야 한다.
- Redis 장애 시 fail-open(허용) 또는 fail-closed(거부) 정책을 미리 결정해야 한다.

**계층별 적용**
- API Gateway 레벨에서 적용하면 서버까지 요청이 도달하지 않아 효율적이다.
- Nginx의 `limit_req_zone`, Kong, AWS API Gateway 등이 내장 기능을 제공한다.

**차별화된 한도**
- 무료/유료 플랜마다 다른 한도를 적용하거나, 엔드포인트별로 다른 정책을 적용하는 경우가 많다.

---

## 알고리즘 비교

| 알고리즘 | 정확도 | 메모리 | 버스트 허용 | 구현 난이도 |
|----------|--------|--------|------------|------------|
| Fixed Window | 낮음 | 낮음 | O | 매우 쉬움 |
| Sliding Window Log | 높음 | 높음 | X | 보통 |
| Token Bucket | 높음 | 낮음 | O (용량만큼) | 보통 |
| Leaky Bucket | 높음 | 보통 | X | 보통 |
| Sliding Window Counter | 높음 | 낮음 | X | 보통 |

실무에서는 **Token Bucket** 또는 **Sliding Window Counter + Redis** 조합이 가장 많이 쓰인다.
