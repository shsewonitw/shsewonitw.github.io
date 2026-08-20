---
layout: post
title: "[Daily morning study] AWS Kinesis Data Streams와 실시간 데이터 스트리밍"
description: >
  #daily morning study
category: 
    - dms
    - dms-cloud
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## AWS Kinesis란

AWS Kinesis는 실시간 스트리밍 데이터를 수집, 처리, 분석하는 완전 관리형 서비스다. 로그, 클릭스트림, IoT 센서 데이터, 금융 거래 이벤트처럼 초당 수백만 건씩 쏟아지는 데이터를 처리할 때 사용한다.

Kinesis 제품군은 크게 4가지다.

| 서비스 | 설명 |
|--------|------|
| Kinesis Data Streams (KDS) | 실시간 스트림 데이터 수집 및 처리 |
| Kinesis Data Firehose | 스트림 → S3, Redshift, OpenSearch 등으로 자동 전달 |
| Kinesis Data Analytics | SQL 또는 Apache Flink로 스트림 데이터 실시간 분석 |
| Kinesis Video Streams | 비디오 스트림 수집 및 처리 |

이 중 가장 핵심적인 Kinesis Data Streams(KDS)를 중심으로 정리한다.

---

## Kinesis Data Streams 핵심 개념

### Stream

KDS에서 데이터를 담는 최상위 논리 단위다. 하나의 Stream은 여러 개의 **Shard**로 구성된다.

### Shard

스트림의 기본 처리 단위다.

- 입력: 초당 1MB 또는 1,000개의 레코드
- 출력: 초당 2MB

Shard 수를 늘리면 그만큼 처리량이 선형으로 증가한다. 처리량이 부족하면 **Shard Splitting**으로 확장하고, 과하면 **Shard Merging**으로 축소한다.

### Record

KDS에 전송되는 데이터 단위다. 세 가지 구성요소를 가진다.

- **Partition Key**: 레코드를 어떤 Shard로 보낼지 결정하는 키
- **Data Blob**: 실제 데이터 (최대 1MB)
- **Sequence Number**: KDS가 자동으로 부여하는 고유 순서 번호

### Retention Period

KDS는 수신된 데이터를 기본 24시간 동안 보관한다. 최대 365일까지 설정할 수 있다. 이 기간 안에 Consumer가 데이터를 읽어가지 않으면 소실된다.

---

## Producer: 데이터 전송

Producer는 스트림에 데이터를 보내는 역할이다. AWS SDK, Kinesis Agent, Kinesis Producer Library(KPL) 등을 통해 데이터를 전송한다.

```python
import boto3
import json

kinesis = boto3.client('kinesis', region_name='ap-northeast-2')

record = {
    "user_id": "u-1234",
    "event": "click",
    "timestamp": "2026-08-15T09:00:00Z"
}

response = kinesis.put_record(
    StreamName='my-stream',
    Data=json.dumps(record),
    PartitionKey='u-1234'   # 같은 user_id는 같은 Shard로 라우팅
)

print(response['ShardId'], response['SequenceNumber'])
```

`PartitionKey`는 동일한 키를 가진 레코드가 항상 같은 Shard로 가도록 보장한다. 이 덕분에 같은 사용자의 이벤트 순서가 보장된다.

### 배치 전송: PutRecords

`put_record` 대신 `put_records`를 쓰면 한 번에 최대 500개의 레코드를 묶어서 전송할 수 있어 네트워크 비용과 지연을 줄일 수 있다.

```python
records = [
    {'Data': json.dumps({'user_id': f'u-{i}', 'event': 'view'}),
     'PartitionKey': f'u-{i}'}
    for i in range(100)
]

response = kinesis.put_records(
    StreamName='my-stream',
    Records=records
)

# 일부 레코드가 실패할 수 있으므로 FailedRecordCount 확인 필요
print(f"Failed: {response['FailedRecordCount']}")
```

---

## Consumer: 데이터 읽기

Consumer는 스트림에서 데이터를 읽어 처리하는 쪽이다. 읽기 방식에는 두 가지가 있다.

### Shared Fan-Out (기본)

여러 Consumer가 Shard의 출력 대역폭(초당 2MB)을 공유한다.

```
Shard 출력 2MB/s ÷ Consumer 수
→ Consumer가 5개면 각각 400KB/s
```

Consumer 수가 늘어날수록 각각의 처리 속도가 낮아진다. 폴링(polling) 방식이라 약 200ms 단위로 데이터를 가져온다.

### Enhanced Fan-Out (향상된 전용 출력)

각 Consumer가 Shard당 2MB/s를 독점적으로 할당받는다. Push 방식이므로 레이턴시가 ~70ms로 낮다. 대신 Consumer당 추가 비용이 발생한다.

```python
# Enhanced Fan-Out 구독 등록
response = kinesis.register_stream_consumer(
    StreamARN='arn:aws:kinesis:...:stream/my-stream',
    ConsumerName='my-consumer'
)
consumer_arn = response['Consumer']['ConsumerARN']
```

### GetRecords로 직접 읽기

```python
# 특정 Shard의 Iterator 획득
shard_iterator = kinesis.get_shard_iterator(
    StreamName='my-stream',
    ShardId='shardId-000000000000',
    ShardIteratorType='LATEST'  # LATEST, TRIM_HORIZON, AT_TIMESTAMP 등
)['ShardIterator']

# 레코드 읽기
while True:
    response = kinesis.get_records(
        ShardIterator=shard_iterator,
        Limit=100
    )
    records = response['Records']
    for record in records:
        data = json.loads(record['Data'])
        print(data)
    
    shard_iterator = response['NextShardIterator']
    if not shard_iterator:
        break
```

### ShardIteratorType 종류

| 타입 | 설명 |
|------|------|
| `LATEST` | 이 시점 이후에 들어오는 새 레코드부터 읽기 |
| `TRIM_HORIZON` | Shard에 보관된 가장 오래된 레코드부터 읽기 |
| `AT_SEQUENCE_NUMBER` | 특정 시퀀스 번호의 레코드부터 읽기 |
| `AFTER_SEQUENCE_NUMBER` | 특정 시퀀스 번호 다음부터 읽기 |
| `AT_TIMESTAMP` | 특정 타임스탬프 이후 레코드부터 읽기 |

---

## Kinesis와 SQS 비교

KDS와 SQS(Simple Queue Service)는 둘 다 메시지 전달 서비스지만 목적이 다르다.

| 항목 | Kinesis Data Streams | SQS |
|------|----------------------|-----|
| 패턴 | 스트림 (순서 보장) | 큐 (기본 순서 미보장) |
| Consumer 수 | 여러 Consumer가 같은 데이터 읽기 가능 | 메시지는 한 Consumer만 처리 |
| 보존 기간 | 최대 365일 | 최대 14일 |
| 처리 순서 | Shard 내에서 순서 보장 | FIFO 큐 사용 시 보장 |
| 주요 사용 사례 | 실시간 분석, 로그 파이프라인 | 작업 큐, 마이크로서비스 간 통신 |

같은 이벤트를 여러 서비스에서 동시에 소비해야 한다면 KDS가 적합하다. 반면 작업 하나를 딱 하나의 Worker만 처리해야 한다면 SQS가 맞다.

---

## Lambda와 연동

KDS를 Lambda의 이벤트 소스로 등록하면 새 레코드가 들어올 때마다 자동으로 Lambda 함수가 트리거된다.

```python
# Lambda 핸들러 예시
def handler(event, context):
    for record in event['Records']:
        # Base64 디코딩 필요
        import base64
        payload = base64.b64decode(record['kinesis']['data'])
        data = json.loads(payload)
        
        print(f"Shard: {record['kinesis']['sequenceNumber']}")
        print(f"Data: {data}")
    
    return {'statusCode': 200}
```

Lambda는 배치 단위로 레코드를 받는다. `BatchSize`와 `BisectBatchOnFunctionError` 설정을 통해 에러 발생 시 배치를 반으로 쪼개 재처리하는 방식으로 부분 실패를 처리할 수 있다.

---

## 실제 아키텍처 예시

실시간 사용자 행동 분석 파이프라인을 구성한다면 이런 형태가 된다.

```
[웹/앱 서버] → (put_record) → [Kinesis Data Streams]
                                      │
              ┌───────────────────────┼──────────────────────┐
              │                       │                      │
    [Lambda: 실시간 알림]    [Lambda: Elasticsearch]   [Firehose → S3]
    (사기 탐지, 푸시 알림)   (검색/분석용 인덱싱)      (배치 분석용 저장)
```

Kinesis의 핵심 가치는 **동일한 스트림 데이터를 여러 Consumer가 독립적으로 처리**할 수 있다는 점이다. 실시간 알림 Lambda가 느려져도 Elasticsearch 인덱싱 Lambda는 영향을 받지 않는다.

---

## 비용 구조

KDS는 Shard 수와 데이터 보존 기간을 기준으로 과금한다.

| 항목 | 기본 단가 (ap-northeast-2 기준) |
|------|-------------------------------|
| Shard 시간 | $0.015 / Shard / 시간 |
| PUT 페이로드 유닛 | $0.014 / 1,000,000 유닛 (25KB당 1유닛) |
| Extended Data Retention (7일 이상) | 추가 과금 |
| Enhanced Fan-Out | $0.015 / Shard / 시간 + $0.013 / GB |

Shard는 미리 프로비저닝하는 방식(Provisioned)과 트래픽에 따라 자동 조정되는 On-Demand 모드 중 선택할 수 있다. On-Demand는 최대 200MB/s까지 자동 확장되지만 단가가 더 비싸다.

---

## 정리

- KDS는 Shard 단위로 데이터를 분산 저장하고 순서를 보장한다
- PartitionKey로 동일한 키의 레코드를 같은 Shard로 라우팅해 순서를 유지한다
- 같은 데이터를 여러 Consumer가 독립적으로 읽을 수 있어 팬아웃 처리에 적합하다
- SQS는 작업 큐, KDS는 스트리밍 파이프라인으로 용도가 다르다
- Lambda, Firehose, Data Analytics와 조합해 실시간 데이터 파이프라인을 구성한다
