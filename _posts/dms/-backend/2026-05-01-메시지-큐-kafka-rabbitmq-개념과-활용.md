---
layout: post
title: "[Daily morning study] 메시지 큐(Kafka, RabbitMQ) 개념과 활용"
description: >
  #daily morning study
category: 
    - dms
    - dms-backend
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---
# 메시지 큐(Kafka, RabbitMQ) 개념과 활용

## 메시지 큐란

메시지 큐(Message Queue)는 서비스 간 비동기 통신을 가능하게 해주는 미들웨어다. 생산자(Producer)가 메시지를 큐에 넣으면, 소비자(Consumer)가 나중에 꺼내서 처리한다. 두 서비스가 동시에 실행되고 있을 필요가 없기 때문에 시스템 간 결합도를 낮추는 데 핵심적인 역할을 한다.

### 왜 필요한가

- 트래픽 급증 시 메시지를 큐에 쌓아두고 소비자가 여유롭게 처리 → **부하 분산**
- 소비자 서비스가 일시 중단돼도 메시지를 잃지 않음 → **내결함성**
- 생산자가 소비자의 응답을 기다리지 않음 → **응답 시간 단축**
- 여러 소비자가 독립적으로 메시지를 처리 → **확장성**

---

## 핵심 개념

| 용어 | 설명 |
|------|------|
| Producer | 메시지를 생성해서 큐/토픽에 보내는 쪽 |
| Consumer | 큐/토픽에서 메시지를 꺼내 처리하는 쪽 |
| Queue / Topic | 메시지가 저장되는 공간 |
| Broker | 메시지를 중개하는 서버 (Kafka, RabbitMQ 자체가 브로커) |
| Acknowledgment (ACK) | 소비자가 메시지를 정상 처리했다고 브로커에 알리는 신호 |

---

## RabbitMQ

### 동작 방식

RabbitMQ는 **AMQP(Advanced Message Queuing Protocol)** 기반의 전통적인 메시지 브로커다. Producer가 메시지를 **Exchange**에 보내면, Exchange가 **바인딩 규칙**에 따라 하나 이상의 Queue로 라우팅한다. Consumer는 Queue에서 메시지를 꺼내 처리한다.

```
Producer → Exchange → (Binding) → Queue → Consumer
```

### Exchange 타입

| 타입 | 설명 |
|------|------|
| Direct | 라우팅 키가 정확히 일치하는 Queue에 전달 |
| Fanout | 연결된 모든 Queue에 브로드캐스트 |
| Topic | 라우팅 키 패턴 매칭 (와일드카드 지원: `*`, `#`) |
| Headers | 헤더 속성 기반 라우팅 |

### 특징

- 메시지를 처리하면 큐에서 **삭제**됨 (소비 후 사라짐)
- ACK를 받아야 큐에서 제거 → ACK 전에 Consumer가 죽으면 **재전송**
- 복잡한 라우팅 로직이 필요할 때 유리
- 실시간 태스크 큐, 이메일 발송, 알림 처리 같은 **단건 처리** 패턴에 적합

### 간단한 Python 예시 (pika 라이브러리)

```python
import pika

# Producer
connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()
channel.queue_declare(queue='task_queue', durable=True)
channel.basic_publish(
    exchange='',
    routing_key='task_queue',
    body='Hello World',
    properties=pika.BasicProperties(delivery_mode=2)  # 메시지 영속성
)
connection.close()

# Consumer
def callback(ch, method, properties, body):
    print(f"Received: {body.decode()}")
    ch.basic_ack(delivery_tag=method.delivery_tag)

channel.basic_qos(prefetch_count=1)
channel.basic_consume(queue='task_queue', on_message_callback=callback)
channel.start_consuming()
```

---

## Apache Kafka

### 동작 방식

Kafka는 **분산 이벤트 스트리밍 플랫폼**으로, 대용량 실시간 데이터 처리를 위해 설계됐다. 메시지는 **Topic**으로 분류되고, Topic은 여러 **Partition**으로 나뉜다. 각 Partition은 순서가 보장된 로그 파일이다.

```
Producer → Topic (Partition 0, 1, 2, ...) → Consumer Group
```

### 핵심 구조

| 개념 | 설명 |
|------|------|
| Topic | 메시지 스트림의 카테고리 |
| Partition | Topic을 분할한 단위, 병렬 처리의 기본 단위 |
| Offset | Partition 내 메시지의 순서 번호 |
| Consumer Group | 여러 Consumer가 협력해 Topic을 분산 처리하는 논리적 그룹 |
| Broker | Kafka 서버 인스턴스, 클러스터로 구성 |
| ZooKeeper / KRaft | 브로커 메타데이터 관리 (KRaft는 ZooKeeper 없이 동작) |

### Partition과 Consumer Group

- 하나의 Partition은 같은 Consumer Group 내 **하나의 Consumer만** 처리
- Consumer Group 수를 늘리면 같은 메시지를 **여러 시스템이 독립적으로** 소비 가능
- Partition 수 = Consumer Group 내 최대 병렬 처리 단위

```
Topic (3 Partitions)
  P0 → Consumer A
  P1 → Consumer B
  P2 → Consumer C
```

### RabbitMQ와의 핵심 차이: 메시지 보존

Kafka는 메시지를 소비해도 **삭제하지 않는다**. 설정한 보존 기간(기본 7일)이 지나거나 디스크 한도에 도달할 때 삭제된다. Consumer는 Offset을 직접 관리하므로, 필요하면 **과거 메시지를 재처리**할 수 있다.

### 특징

- 초당 수백만 건의 메시지 처리 가능 → **고처리량**
- 메시지를 디스크에 순차 기록 → **내구성**
- 이벤트 소싱, 로그 수집, 실시간 분석 파이프라인에 적합
- 설정과 운영 복잡도가 RabbitMQ보다 높음

### 간단한 Python 예시 (kafka-python 라이브러리)

```python
from kafka import KafkaProducer, KafkaConsumer
import json

# Producer
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)
producer.send('user-events', {'user_id': 42, 'action': 'login'})
producer.flush()

# Consumer
consumer = KafkaConsumer(
    'user-events',
    bootstrap_servers=['localhost:9092'],
    group_id='analytics-group',
    value_deserializer=lambda m: json.loads(m.decode('utf-8'))
)
for message in consumer:
    print(f"Offset {message.offset}: {message.value}")
```

---

## RabbitMQ vs Kafka 비교

| 항목 | RabbitMQ | Kafka |
|------|----------|-------|
| 프로토콜 | AMQP | 자체 바이너리 프로토콜 |
| 메시지 보존 | 소비 후 삭제 | 설정 기간 보존 |
| 처리량 | 중간 (수만 건/초) | 매우 높음 (수백만 건/초) |
| 순서 보장 | Queue 단위 | Partition 단위 |
| 라우팅 | Exchange를 통한 유연한 라우팅 | Topic + Partition 기반 |
| 재처리 | 기본 어려움 (DLQ 활용 가능) | Offset 재설정으로 용이 |
| 적합한 용도 | 태스크 큐, 복잡한 라우팅 | 이벤트 스트리밍, 로그 파이프라인 |
| 운영 복잡도 | 낮음 | 높음 |

---

## 실무 선택 기준

**RabbitMQ를 선택하는 경우**
- 이메일 발송, 결제 처리처럼 각 메시지를 **정확히 한 번** 처리해야 할 때
- 복잡한 라우팅 규칙이 필요할 때
- 팀이 빠르게 도입하고 싶을 때

**Kafka를 선택하는 경우**
- 사용자 행동 로그, 클릭스트림처럼 **대용량 이벤트**를 실시간으로 처리해야 할 때
- 여러 시스템이 같은 이벤트를 **독립적으로 소비**해야 할 때
- 과거 데이터를 **재처리**하거나 이벤트 소싱 패턴을 구현할 때

---

## 메시지 전달 보장 수준

| 수준 | 설명 |
|------|------|
| At-most-once | 메시지가 유실될 수 있지만 중복 없음 |
| At-least-once | 메시지가 반드시 전달되지만 중복 가능 |
| Exactly-once | 정확히 한 번 전달 (구현 복잡, 성능 저하) |

Kafka는 **At-least-once**가 기본이며, Idempotent Producer + Transactional API를 활용하면 Exactly-once도 가능하다. RabbitMQ는 ACK 기반으로 At-least-once를 보장한다.
