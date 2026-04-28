---
layout: post
title: "[Daily morning study] AWS SQS, SNS 차이점과 활용"
description: >
  #daily morning study
category: 
    - dms
    - dms-cloud
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---
# AWS SQS, SNS 차이점과 활용

AWS에서 서비스 간 비동기 통신을 구현할 때 가장 먼저 마주치는 두 가지 서비스가 SQS와 SNS다. 이름이 비슷하고 함께 쓰이는 경우가 많지만, 설계 목적과 동작 방식이 다르다.

---

## SQS (Simple Queue Service)

SQS는 **메시지 큐** 서비스다. 생산자(Producer)가 메시지를 큐에 넣으면, 소비자(Consumer)가 큐에서 꺼내 처리하는 구조다.

### 핵심 특징

- **Pull 방식**: 소비자가 직접 큐를 폴링해서 메시지를 가져간다.
- **1:1 통신**: 하나의 메시지는 하나의 소비자만 처리한다. 꺼내가면 큐에서 삭제된다.
- **비동기 버퍼**: 생산자와 소비자의 처리 속도가 달라도 괜찮다. 큐가 완충 역할을 한다.
- **재처리 보장**: 메시지를 처리한 후 명시적으로 삭제하지 않으면 다시 가시화된다(visibility timeout).

### 큐 종류

| 구분 | Standard Queue | FIFO Queue |
|------|---------------|------------|
| 순서 보장 | 보장 안 됨 (Best-effort) | 보장됨 |
| 중복 | 가끔 발생 가능 | 정확히 1회 처리 |
| 처리량 | 무제한 (초당 수만 TPS) | 초당 최대 3,000 TPS |
| 사용 예 | 이미지 리사이징, 이메일 발송 | 결제 처리, 주문 시스템 |

### Dead Letter Queue (DLQ)

처리 실패한 메시지를 별도 큐로 보내는 기능이다. 특정 횟수(maxReceiveCount)를 초과하면 DLQ로 이동시켜 나중에 분석하거나 재처리할 수 있다.

```
메시지 → SQS 큐 → Lambda (처리 실패 3회) → DLQ
```

### 예시: Lambda와 SQS 연동

```python
import boto3
import json

sqs = boto3.client('sqs', region_name='ap-northeast-2')
queue_url = 'https://sqs.ap-northeast-2.amazonaws.com/123456789/my-queue'

# 메시지 전송
sqs.send_message(
    QueueUrl=queue_url,
    MessageBody=json.dumps({'order_id': 42, 'item': 'keyboard'})
)

# 메시지 수신 (폴링)
response = sqs.receive_message(
    QueueUrl=queue_url,
    MaxNumberOfMessages=10,
    WaitTimeSeconds=20  # Long polling
)

for msg in response.get('Messages', []):
    body = json.loads(msg['Body'])
    print(body)
    # 처리 완료 후 삭제
    sqs.delete_message(
        QueueUrl=queue_url,
        ReceiptHandle=msg['ReceiptHandle']
    )
```

Short polling은 즉시 응답하지만 빈 응답이 많아 비용이 높아진다. `WaitTimeSeconds`를 설정한 Long polling을 쓰면 메시지가 올 때까지 최대 20초 대기하므로 효율적이다.

---

## SNS (Simple Notification Service)

SNS는 **발행-구독(Pub/Sub)** 서비스다. 발행자(Publisher)가 토픽에 메시지를 보내면, 구독 중인 모든 엔드포인트에 동시에 전달된다.

### 핵심 특징

- **Push 방식**: 메시지를 구독자에게 능동적으로 밀어넣는다.
- **1:N 팬아웃**: 하나의 메시지를 여러 구독자가 동시에 받는다.
- **메시지 저장 없음**: 전달 후 메시지를 보관하지 않는다. 구독자가 오프라인이면 수신 불가.
- **다양한 구독 타입**: SQS, Lambda, HTTP/S 엔드포인트, 이메일, SMS, 모바일 푸시 등을 구독자로 등록할 수 있다.

### 예시: SNS 토픽에 메시지 발행

```python
import boto3
import json

sns = boto3.client('sns', region_name='ap-northeast-2')
topic_arn = 'arn:aws:sns:ap-northeast-2:123456789:order-events'

sns.publish(
    TopicArn=topic_arn,
    Message=json.dumps({'event': 'ORDER_PLACED', 'order_id': 42}),
    Subject='New Order'
)
```

메시지를 한 번 발행하면 토픽을 구독한 Lambda, SQS, 이메일 수신자 모두에게 동시에 전달된다.

---

## SQS vs SNS 비교

| 항목 | SQS | SNS |
|------|-----|-----|
| 통신 모델 | 큐(Queue) | 발행-구독(Pub/Sub) |
| 수신 방식 | Pull (소비자 폴링) | Push (능동 전달) |
| 수신자 수 | 1개 (한 소비자만) | N개 (모든 구독자) |
| 메시지 보존 | 최대 14일 | 없음 (전달 후 소멸) |
| 사용 목적 | 작업 분산, 버퍼링 | 이벤트 브로드캐스팅 |
| 재처리 | DLQ로 재처리 가능 | 기본 미지원 |

---

## SNS + SQS 팬아웃 패턴

실전에서는 SNS와 SQS를 조합해서 쓰는 경우가 많다. 이를 **팬아웃(Fanout)** 패턴이라 한다.

```
                         ┌─> SQS (주문 처리 서비스)
Publisher → SNS Topic ──┼─> SQS (재고 관리 서비스)
                         └─> Lambda (이메일 알림)
```

예를 들어 주문이 발생하면:
1. SNS 토픽에 `ORDER_PLACED` 이벤트를 발행한다.
2. 주문 처리 SQS, 재고 관리 SQS, 알림 Lambda가 동시에 이벤트를 받는다.
3. 각 서비스는 독립적으로 처리한다.

직접 각 서비스에 메시지를 보내면 생산자가 소비자를 알아야 하지만, 팬아웃 패턴에서는 생산자와 소비자가 완전히 분리된다.

---

## 언제 무엇을 쓸까

**SQS만 쓰는 경우**
- 작업 큐가 필요할 때: 이미지 처리, 이메일 발송, 비디오 인코딩 등 처리 속도 차이를 완충해야 하는 상황
- 정확히 한 번 처리를 보장해야 할 때 (FIFO 큐)

**SNS만 쓰는 경우**
- 여러 시스템에 동시에 알림을 보낼 때: 모바일 푸시 알림, SMS, 이메일 브로드캐스트

**SNS + SQS 조합**
- MSA에서 이벤트 기반 아키텍처를 구성할 때
- 하나의 이벤트를 여러 서비스가 독립적으로 처리해야 할 때
- 소비자 서비스의 처리 속도가 다를 때 (SQS가 버퍼 역할)

---

## 메시지 필터링

SNS는 구독자별로 메시지 필터 정책을 설정할 수 있다. 특정 속성을 가진 메시지만 수신하도록 제한하면 불필요한 처리를 줄일 수 있다.

```json
{
  "event_type": ["ORDER_PLACED", "ORDER_CANCELLED"]
}
```

위 정책을 가진 구독자는 `event_type`이 `ORDER_PLACED` 또는 `ORDER_CANCELLED`인 메시지만 받는다. 다른 이벤트는 해당 구독자에게 전달되지 않는다.

---

## 정리

- **SQS**: 작업 큐. Pull 방식. 메시지를 하나의 소비자가 처리. 속도 차이 완충.
- **SNS**: 이벤트 브로드캐스트. Push 방식. 모든 구독자에게 동시 전달.
- **팬아웃 패턴**: SNS → 여러 SQS로 이벤트를 분산. MSA에서 서비스 간 결합도를 낮추는 핵심 패턴.
