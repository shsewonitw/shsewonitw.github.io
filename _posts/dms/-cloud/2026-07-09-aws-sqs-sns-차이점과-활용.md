---
layout: post
title: "[Daily morning study] AWS SQS와 SNS 차이점과 활용"
description: >
  #daily morning study
category: 
    - dms
    - dms-cloud
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## AWS SQS와 SNS

클라우드 아키텍처에서 서비스 간 통신을 느슨하게 결합(loosely coupled)하기 위해 AWS에서는 SQS와 SNS를 자주 사용한다. 둘 다 메시지 기반 서비스지만 동작 방식과 목적이 다르다.

---

## SQS (Simple Queue Service)

SQS는 **메시지 큐** 서비스다. 프로듀서가 큐에 메시지를 넣고, 컨슈머가 큐에서 메시지를 가져가서 처리하는 **Point-to-Point** 방식으로 동작한다.

### 핵심 동작 방식

1. 프로듀서가 SQS 큐에 메시지를 전송
2. 메시지는 큐에 보관됨 (기본 최대 4일, 최대 14일)
3. 컨슈머가 큐를 **폴링(polling)** 해서 메시지를 가져옴
4. 처리 완료 후 컨슈머가 메시지를 **명시적으로 삭제**

컨슈머가 메시지를 가져간 순간부터 **Visibility Timeout** 시간 동안 다른 컨슈머에게는 해당 메시지가 보이지 않는다. 처리에 실패하면 Visibility Timeout이 지난 뒤 큐에 다시 나타난다.

### SQS 큐 종류

| 구분 | Standard Queue | FIFO Queue |
|------|---------------|------------|
| 처리량 | 무제한 (초당 수만 건) | 초당 300건 (배치 시 3,000건) |
| 순서 보장 | 보장 안 됨 (최선) | 완전 보장 |
| 중복 가능성 | 가끔 중복 발생 | 중복 없음 |
| 사용 사례 | 처리량 우선, 순서 무관 작업 | 금융 거래, 주문 처리 등 순서 중요한 작업 |

### Dead Letter Queue (DLQ)

처리에 계속 실패하는 메시지를 별도 큐(DLQ)로 분리해서 추후 분석할 수 있다. `maxReceiveCount`를 초과하면 자동으로 DLQ로 이동된다.

### 주요 설정 값

- **Message Retention Period**: 메시지 보관 기간 (기본 4일, 최대 14일)
- **Visibility Timeout**: 컨슈머가 처리하는 동안 다른 컨슈머에게 숨기는 시간 (기본 30초)
- **Message Size**: 최대 256KB (더 큰 경우 S3에 저장 후 포인터만 전달)
- **Long Polling**: 빈 큐 폴링 시 최대 20초 대기해서 빈 응답 줄임

---

## SNS (Simple Notification Service)

SNS는 **Pub/Sub(발행-구독) 메시징 서비스**다. 프로듀서가 SNS 토픽에 메시지를 발행하면, 해당 토픽을 구독하는 **모든 구독자에게 동시에 전달(push)**된다.

### 핵심 동작 방식

1. 프로듀서가 SNS 토픽에 메시지 발행
2. SNS가 구독 중인 모든 엔드포인트에 **즉시 푸시**
3. 메시지는 큐에 보관되지 않음 (전달 후 소멸)

### 지원 구독 프로토콜

| 프로토콜 | 설명 |
|----------|------|
| SQS | SQS 큐로 메시지 전달 |
| Lambda | Lambda 함수 직접 호출 |
| HTTP/HTTPS | 지정된 엔드포인트로 POST 요청 |
| Email | 이메일 발송 |
| SMS | 문자 메시지 발송 |
| Mobile Push | FCM, APNS 등 모바일 푸시 알림 |

---

## SQS vs SNS 비교

| 항목 | SQS | SNS |
|------|-----|-----|
| 메시징 패턴 | Point-to-Point | Pub/Sub |
| 전달 방식 | 컨슈머가 폴링 (Pull) | SNS가 구독자에게 푸시 (Push) |
| 수신자 수 | 하나의 컨슈머가 메시지 처리 | 여러 구독자에게 동시 전달 |
| 메시지 보관 | 큐에 보관 (최대 14일) | 보관 없음 (전달 즉시 소멸) |
| 주 사용 목적 | 비동기 작업 큐, 부하 분산 | 이벤트 브로드캐스트, 알림 |

---

## SNS + SQS Fan-out 패턴

실무에서는 SNS와 SQS를 함께 쓰는 **Fan-out 패턴**이 자주 등장한다.

```
[Producer]
    |
    ↓
[SNS Topic]
    |
    ├──→ [SQS Queue A] → Consumer A (재고 감소)
    ├──→ [SQS Queue B] → Consumer B (주문 확인 이메일 발송)
    └──→ [SQS Queue C] → Consumer C (통계/분석 처리)
```

**장점:**
- 프로듀서는 SNS에만 발행하면 되므로 구독자 추가/제거에 영향을 받지 않음
- 각 SQS 큐가 독립적으로 메시지를 처리하므로 한 컨슈머가 느려도 다른 컨슈머에 영향 없음
- SQS가 버퍼 역할을 해서 컨슈머 처리 속도에 맞게 소비 가능

### 구현 예시 (AWS SDK for Python)

```python
import boto3
import json

sns = boto3.client('sns', region_name='ap-northeast-2')
sqs = boto3.client('sqs', region_name='ap-northeast-2')

# SNS 토픽에 메시지 발행
def publish_order_event(order_id: str, user_id: str):
    message = {
        "order_id": order_id,
        "user_id": user_id,
        "event": "ORDER_COMPLETED"
    }
    response = sns.publish(
        TopicArn='arn:aws:sns:ap-northeast-2:123456789012:order-events',
        Message=json.dumps(message),
        Subject='OrderCompleted'
    )
    return response['MessageId']

# SQS 큐에서 메시지 수신 및 처리
def process_inventory_queue(queue_url: str):
    response = sqs.receive_message(
        QueueUrl=queue_url,
        MaxNumberOfMessages=10,
        WaitTimeSeconds=20  # Long Polling
    )
    messages = response.get('Messages', [])
    for msg in messages:
        body = json.loads(msg['Body'])
        # SNS → SQS 전달 시 body 안에 실제 메시지가 중첩됨
        sns_message = json.loads(body['Message'])
        print(f"재고 처리: {sns_message['order_id']}")

        # 처리 완료 후 메시지 삭제
        sqs.delete_message(
            QueueUrl=queue_url,
            ReceiptHandle=msg['ReceiptHandle']
        )
```

---

## 실무 적용 시 고려 사항

**SQS를 단독으로 쓰는 경우**
- 작업을 특정 워커 하나가 처리해야 할 때 (e.g., 이미지 리사이징, 이메일 발송 큐)
- 처리 속도보다 프로듀서 발행 속도가 빨라서 버퍼가 필요할 때

**SNS를 단독으로 쓰는 경우**
- 실시간 알림, SMS, 이메일 등 전달만 하면 되는 간단한 경우
- 재처리나 지연 처리가 필요 없을 때

**SNS + SQS Fan-out을 쓰는 경우**
- 하나의 이벤트가 여러 다운스트림 서비스에 영향을 줘야 할 때
- 각 서비스가 독립적인 처리 속도와 재시도 로직을 가져야 할 때
- 마이크로서비스 간 느슨한 결합이 필요할 때

**메시지 필터링 (SNS Filter Policy)**

SNS는 구독자별로 필터 정책을 설정해서 특정 속성의 메시지만 수신할 수 있다.

```json
{
  "event": ["ORDER_COMPLETED", "ORDER_CANCELLED"],
  "region": ["KR"]
}
```

위 정책을 가진 SQS 큐는 `event`가 `ORDER_COMPLETED` 또는 `ORDER_CANCELLED`이고 `region`이 `KR`인 메시지만 수신한다. 이를 통해 불필요한 메시지 처리를 줄일 수 있다.
