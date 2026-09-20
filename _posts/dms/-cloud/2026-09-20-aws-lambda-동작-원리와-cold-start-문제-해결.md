---
layout: post
title: "[Daily morning study] AWS Lambda 동작 원리와 Cold Start 문제 해결"
description: >
  #daily morning study
category: 
    - dms
    - dms-cloud
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## AWS Lambda 동작 원리

Lambda는 이벤트 기반의 서버리스 컴퓨팅 서비스다. 코드를 배포하면 AWS가 실행 환경(Execution Environment)을 관리하고, 요청이 들어올 때만 함수를 실행한다.

### 실행 환경(Execution Environment) 생명주기

Lambda 함수가 호출될 때 내부적으로 세 단계를 거친다.

| 단계 | 설명 |
|------|------|
| Init | 실행 환경 생성, 코드 다운로드, 런타임 초기화, 핸들러 외부 코드 실행 |
| Invoke | 핸들러 함수 실행 |
| Shutdown | 실행 환경 정리 및 해제 |

Init 단계는 첫 호출 또는 실행 환경이 없을 때만 발생한다. 이미 준비된 실행 환경이 있으면 Invoke 단계로 바로 진입한다.

### 실행 환경 재사용

Lambda는 연속된 요청에 대해 동일한 실행 환경을 재사용한다(Warm Start). 핸들러 함수 외부에서 초기화한 객체(DB 커넥션, SDK 클라이언트 등)는 재사용 요청에서 그대로 살아있다.

```python
import boto3

# 핸들러 외부 — 실행 환경 재사용 시 재초기화하지 않음
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Users')

def handler(event, context):
    # 핸들러 내부 — 매 요청마다 실행
    response = table.get_item(Key={'userId': event['userId']})
    return response['Item']
```

이 패턴 덕분에 커넥션 초기화 비용을 요청마다 지불하지 않아도 된다.

---

## Cold Start 문제

### Cold Start란

실행 환경이 없는 상태에서 Lambda를 처음 호출할 때 Init 단계가 실행되면서 응답 지연이 발생하는 현상이다. 일반적으로 수백 ms ~ 수 초의 지연이 생긴다.

**Cold Start가 발생하는 경우:**
- 함수를 처음 배포하고 호출할 때
- 트래픽이 없어 실행 환경이 회수된 후
- 동시 요청 수가 기존 실행 환경 수를 초과할 때
- 함수 코드나 설정을 변경 후 첫 호출 시

### Cold Start 지연 요소

```
[Cold Start 지연 구성]
VPC 환경 설정 (ENI 생성)  → 수백 ms ~ 1s
런타임 초기화 (JVM, .NET) → 수백 ms
코드 패키지 다운로드      → 패키지 크기에 비례
핸들러 외부 코드 실행     → 초기화 로직에 따라 다름
```

Java나 .NET처럼 무거운 런타임은 Python, Node.js보다 Cold Start 시간이 훨씬 길다.

---

## Cold Start 해결 방법

### 1. Provisioned Concurrency (프로비저닝된 동시성)

지정한 수만큼 실행 환경을 미리 초기화해 두는 기능이다. Cold Start 없이 즉시 Invoke 단계로 진입한다.

```bash
# 특정 함수 버전에 프로비저닝된 동시성 설정
aws lambda put-provisioned-concurrency-config \
  --function-name MyFunction \
  --qualifier 1 \
  --provisioned-concurrent-executions 10
```

- 비용: 프로비저닝된 환경이 유휴 상태여도 요금이 발생한다.
- Auto Scaling과 연동해 트래픽 패턴에 따라 프로비저닝 수를 자동으로 조절할 수 있다.

### 2. 런타임 선택

| 런타임 | Cold Start 특성 |
|--------|----------------|
| Python, Node.js | 빠른 초기화, 경량 런타임 |
| Java (with GraalVM) | GraalVM 네이티브 이미지로 개선 가능 |
| Java (with SnapStart) | Lambda SnapStart로 최대 10배 단축 |
| .NET | .NET 8 Native AOT로 개선 가능 |

**Lambda SnapStart (Java 전용)**  
함수 초기화 후 메모리 스냅샷을 생성해두고, 이후 호출 시 스냅샷에서 복원하는 방식이다. Init 단계를 건너뛰는 효과를 낸다.

### 3. 패키지 크기 최소화

코드 다운로드 시간을 줄이려면 패키지를 경량화해야 한다.

```python
# 나쁜 예: 사용하지 않는 의존성 포함
requirements.txt:
  boto3       # Lambda 실행 환경에 이미 포함됨
  numpy
  pandas
  scipy       # 실제로는 미사용

# 좋은 예: 실제 필요한 것만 포함
requirements.txt:
  numpy
  pandas
```

- 배포 패키지 권장 크기: 압축 기준 50MB 이하
- Lambda Layer를 활용해 공통 의존성을 분리하면 함수 코드만 배포할 수 있다.

### 4. 핸들러 외부 초기화 최적화

```python
import boto3
import os

# 환경 변수로 DB 접속 정보 관리
DB_HOST = os.environ['DB_HOST']

# 글로벌 스코프에서 한 번만 초기화
connection = create_db_connection(DB_HOST)

def handler(event, context):
    # 이미 초기화된 connection 재사용
    result = connection.query(...)
    return result
```

핸들러 외부에 무거운 초기화 로직을 배치하면 Warm Start 시에는 이 비용이 0이 되지만, Cold Start 지연은 그만큼 늘어난다. 균형이 필요하다.

### 5. VPC 설정 최소화

Lambda를 VPC 안에 배치하면 ENI(Elastic Network Interface) 생성 시간이 추가된다. VPC가 꼭 필요하지 않다면 VPC 밖에 두는 것이 Cold Start에 유리하다.

VPC가 필요한 경우에는 `Hyperplane ENI` 방식(2019년 이후 적용)으로 이미 ENI 생성 지연이 크게 개선되었다.

---

## 동시성(Concurrency) 관리

### 기본 동시성 모델

Lambda는 요청마다 별도의 실행 환경에서 처리된다. 동시에 100개 요청이 오면 100개의 실행 환경이 필요하다.

```
요청 1 → [실행 환경 A]
요청 2 → [실행 환경 B]  (동시 처리)
요청 3 → [실행 환경 C]  (동시 처리)
```

### Reserved Concurrency (예약된 동시성)

특정 함수에 동시성 한도를 설정해 다른 함수가 계정 전체 동시성 풀을 모두 소비하지 못하도록 막는다.

```bash
aws lambda put-function-concurrency \
  --function-name CriticalFunction \
  --reserved-concurrent-executions 100
```

0으로 설정하면 함수 실행을 완전히 막을 수 있다(배포 롤백, 비상 차단 용도).

---

## 실행 환경 재사용 시 주의사항

실행 환경이 재사용될 때 `/tmp` 디렉토리에 남긴 임시 파일이나 전역 변수 상태가 다음 요청에서도 살아있다.

```python
import random

# 잘못된 패턴: 전역 변수가 요청 간 공유됨
request_count = 0

def handler(event, context):
    global request_count
    request_count += 1  # 다른 사용자 요청에서 공유되는 상태
    return {'count': request_count}  # 의도하지 않은 값 반환 가능
```

요청별로 독립적이어야 하는 상태는 반드시 핸들러 내부에서 초기화해야 한다.

---

## 정리

| 문제 | 해결책 |
|------|--------|
| Cold Start 지연 전반 | Provisioned Concurrency |
| Java 런타임 Cold Start | Lambda SnapStart |
| 패키지 초기화 시간 | 패키지 경량화, Lambda Layer |
| VPC 연결 지연 | VPC 제거 또는 Hyperplane ENI 활용 |
| 트래픽 급증 | Auto Scaling + Provisioned Concurrency |

Lambda의 Cold Start는 서버리스 아키텍처의 본질적인 트레이드오프다. 항상 최소화해야 하는 게 아니라, 워크로드 특성(지연 민감도, 비용, 트래픽 패턴)에 맞게 적절한 전략을 선택하는 것이 중요하다.
