---
layout: post
title: "[Daily morning study] Redis Pub/Sub과 Stream 활용"
description: >
  #daily morning study
category: 
    - dms
    - dms-database
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---
# Redis Pub/Sub과 Stream 활용

## Pub/Sub이란

Pub/Sub(Publish/Subscribe)은 메시지를 발행(publish)하는 쪽과 구독(subscribe)하는 쪽을 분리하는 메시징 패턴이다.

- **Publisher**: 특정 채널에 메시지를 보낸다
- **Subscriber**: 채널을 구독하고, 메시지가 오면 즉시 받는다
- Redis가 중간 브로커 역할을 한다

```
Publisher → [Redis Channel] → Subscriber 1
                           → Subscriber 2
                           → Subscriber N
```

## Redis Pub/Sub 기본 명령어

```bash
# 채널 구독
SUBSCRIBE chat-room

# 패턴으로 구독 (glob 패턴 지원)
PSUBSCRIBE chat-*

# 메시지 발행
PUBLISH chat-room "안녕하세요"

# 현재 활성 채널 목록
PUBSUB CHANNELS

# 채널의 구독자 수
PUBSUB NUMSUB chat-room
```

## Pub/Sub의 한계

Redis Pub/Sub은 **fire-and-forget** 방식이다. 이 때문에 몇 가지 제약이 있다.

| 한계 | 설명 |
|------|------|
| 메시지 유실 | 구독자가 오프라인일 때 발행된 메시지는 사라진다 |
| 재전송 불가 | 한 번 발행된 메시지를 다시 받을 수 없다 |
| 소비 확인 없음 | 구독자가 실제로 처리했는지 알 수 없다 |
| 메시지 영속성 없음 | Redis 재시작 시 모두 소실된다 |

이러한 한계를 극복하기 위해 나온 것이 **Redis Stream**이다.

## Redis Stream이란

Redis 5.0에서 추가된 자료구조로, Kafka와 유사한 append-only log 방식의 메시지 스트림이다.

- 메시지가 디스크에 영속적으로 저장된다
- 소비자 그룹(Consumer Group)을 통해 메시지 처리 상태를 추적할 수 있다
- 소비자가 오프라인이었다가 다시 연결해도 놓친 메시지를 읽을 수 있다

## Stream 기본 명령어

```bash
# 메시지 추가 (ID는 자동 생성: 타임스탬프-시퀀스)
XADD orders * product "laptop" price "1200"
# 결과: "1716710400000-0"

# 범위 조회 (처음부터 끝까지)
XRANGE orders - +

# 최신 메시지부터 역순 조회
XREVRANGE orders + - COUNT 5

# 스트림 길이
XLEN orders

# 특정 ID 이후 메시지 읽기 (새 메시지 대기, 블로킹)
XREAD COUNT 10 BLOCK 0 STREAMS orders 1716710400000-0
```

## Consumer Group

여러 소비자가 같은 스트림을 분산 처리할 때 사용한다. 각 메시지는 그룹 내 하나의 소비자에게만 전달된다.

```bash
# Consumer Group 생성 (0부터 시작 = 처음부터)
XGROUP CREATE orders order-group $ MKSTREAM

# Consumer Group으로 메시지 읽기
XREADGROUP GROUP order-group consumer-1 COUNT 5 STREAMS orders >

# 메시지 처리 완료 확인 (ACK)
XACK orders order-group 1716710400000-0

# 처리 중인(ACK 안 된) 메시지 목록
XPENDING orders order-group - + 10
```

`>` 기호는 아직 아무 소비자에게도 전달되지 않은 새 메시지를 의미한다.

## Pub/Sub vs Stream 비교

| 항목 | Pub/Sub | Stream |
|------|---------|--------|
| 메시지 유지 | X (전달 후 사라짐) | O (append-only 로그) |
| 오프라인 소비자 | 메시지 유실 | 재연결 후 재처리 가능 |
| 소비자 그룹 | X | O |
| ACK 지원 | X | O |
| 재생(Replay) | X | O |
| 사용 사례 | 실시간 알림, 채팅 | 이벤트 로그, 주문 처리 |

## Stream 트리밍

Stream은 계속 쌓이므로 크기를 제한해야 할 때가 있다.

```bash
# 최대 1000개 유지 (초과분 자동 삭제)
XADD orders MAXLEN 1000 * product "mouse" price "30"

# 명시적 트리밍
XTRIM orders MAXLEN 1000
```

`MAXLEN ~` (틸다 사용)으로 근사값 트리밍을 하면 성능이 더 좋다.

```bash
XADD orders MAXLEN ~ 1000 * product "keyboard" price "80"
```

## 실사용 시나리오

### 실시간 채팅 → Pub/Sub 적합

```python
import redis

r = redis.Redis()

# 채팅 서버
def send_message(channel, message):
    r.publish(channel, message)

# 채팅 클라이언트
def listen(channel):
    pubsub = r.pubsub()
    pubsub.subscribe(channel)
    for message in pubsub.listen():
        if message['type'] == 'message':
            print(message['data'])
```

### 주문 이벤트 처리 → Stream 적합

```python
import redis

r = redis.Redis()

# 주문 발행
def place_order(product, price):
    r.xadd('orders', {'product': product, 'price': str(price)})

# 주문 처리 워커
def process_orders():
    r.xgroup_create('orders', 'order-group', '$', mkstream=True)
    while True:
        messages = r.xreadgroup('order-group', 'worker-1', {'orders': '>'}, count=10)
        for stream, msgs in messages:
            for msg_id, data in msgs:
                print(f"처리 중: {data}")
                # 처리 완료 시 ACK
                r.xack('orders', 'order-group', msg_id)
```

## 정리

- **Pub/Sub**: 빠르고 단순하지만 메시지가 휘발된다. 실시간 알림이나 캐시 무효화 브로드캐스트에 적합
- **Stream**: 메시지 영속성과 소비자 그룹을 지원해 Kafka 같은 사용 사례에 적합하지만, Kafka만큼 확장성이 높지는 않다
- 메시지 유실이 허용되지 않는 서비스라면 Pub/Sub 대신 Stream을 쓰는 게 맞다
