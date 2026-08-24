---
layout: post
title: "[Daily morning study] AWS Step Functions와 서버리스 워크플로우 오케스트레이션"
description: >
  #daily morning study
category: 
    - dms
    - dms-cloud
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## AWS Step Functions란

AWS Step Functions는 여러 AWS 서비스(Lambda, ECS, DynamoDB, SNS, SQS 등)를 시각적으로 연결해서 워크플로우를 구성하는 완전 관리형 오케스트레이션 서비스다. 각 단계(State)를 JSON 기반의 Amazon States Language(ASL)로 정의하고, 실패·재시도·분기·병렬 처리를 선언적으로 표현할 수 있다.

**핵심 개념**

| 용어 | 설명 |
|------|------|
| State Machine | 전체 워크플로우 정의. 여러 State의 집합 |
| State | 워크플로우의 개별 단계 (Task, Choice, Wait, Parallel 등) |
| Execution | State Machine의 실제 실행 인스턴스 |
| ASL | Amazon States Language. State Machine을 정의하는 JSON 포맷 |

---

## State 유형

### Task State
Lambda, ECS, DynamoDB 등 실제 작업을 수행하는 상태다.

```json
"ProcessOrder": {
  "Type": "Task",
  "Resource": "arn:aws:lambda:ap-northeast-2:123456789012:function:ProcessOrderFn",
  "Next": "NotifyCustomer",
  "Retry": [
    {
      "ErrorEquals": ["Lambda.ServiceException"],
      "IntervalSeconds": 2,
      "MaxAttempts": 3,
      "BackoffRate": 2
    }
  ],
  "Catch": [
    {
      "ErrorEquals": ["States.ALL"],
      "Next": "HandleError"
    }
  ]
}
```

### Choice State
조건에 따라 분기 처리를 담당한다. `if/else` 와 같은 역할이다.

```json
"CheckPayment": {
  "Type": "Choice",
  "Choices": [
    {
      "Variable": "$.paymentStatus",
      "StringEquals": "SUCCESS",
      "Next": "ShipOrder"
    },
    {
      "Variable": "$.paymentStatus",
      "StringEquals": "FAILED",
      "Next": "RefundOrder"
    }
  ],
  "Default": "WaitForPayment"
}
```

### Parallel State
여러 작업을 동시에 실행하고, 모두 완료된 후 다음 단계로 넘어간다.

```json
"ParallelProcessing": {
  "Type": "Parallel",
  "Branches": [
    {
      "StartAt": "SendEmail",
      "States": {
        "SendEmail": { "Type": "Task", "Resource": "...", "End": true }
      }
    },
    {
      "StartAt": "UpdateInventory",
      "States": {
        "UpdateInventory": { "Type": "Task", "Resource": "...", "End": true }
      }
    }
  ],
  "Next": "CompleteOrder"
}
```

### Wait State
지정한 시간 동안 대기하거나 특정 시각까지 기다린다.

```json
"WaitForApproval": {
  "Type": "Wait",
  "Seconds": 300,
  "Next": "CheckApproval"
}
```

### Map State
배열의 각 항목에 대해 동일한 작업을 반복 실행한다. `forEach` 와 비슷하다.

```json
"ProcessItems": {
  "Type": "Map",
  "ItemsPath": "$.orders",
  "MaxConcurrency": 5,
  "Iterator": {
    "StartAt": "ProcessSingleOrder",
    "States": {
      "ProcessSingleOrder": { "Type": "Task", "Resource": "...", "End": true }
    }
  },
  "Next": "Done"
}
```

---

## Standard vs Express 워크플로우

두 가지 실행 유형이 있으며, 사용 사례에 따라 선택해야 한다.

| 항목 | Standard | Express |
|------|----------|---------|
| 최대 실행 시간 | 1년 | 5분 |
| 실행 보장 | Exactly-once | At-least-once |
| 실행 내역 보관 | 90일 | CloudWatch Logs |
| 가격 기준 | 상태 전환 횟수 | 실행 수 + 실행 시간 |
| 주 사용 사례 | 주문 처리, 장기 승인 워크플로우 | 고처리량 이벤트 처리, IoT 데이터 파이프라인 |

Express는 초당 수십만 건의 고처리량 시나리오에 적합하고, Standard는 정확성이 중요한 비즈니스 프로세스에 적합하다.

---

## 오류 처리 패턴

### Retry와 Catch

Step Functions는 Task 레벨에서 자동 재시도와 오류 포착을 지원한다.

```json
"CallExternalAPI": {
  "Type": "Task",
  "Resource": "arn:aws:lambda:...",
  "Retry": [
    {
      "ErrorEquals": ["States.Timeout", "Lambda.AWSLambdaException"],
      "IntervalSeconds": 1,
      "MaxAttempts": 3,
      "BackoffRate": 2.0,
      "JitterStrategy": "FULL"
    }
  ],
  "Catch": [
    {
      "ErrorEquals": ["States.ALL"],
      "ResultPath": "$.errorInfo",
      "Next": "FallbackState"
    }
  ],
  "Next": "Success"
}
```

- `BackoffRate`: 재시도 간격이 지수적으로 증가하는 비율
- `JitterStrategy: FULL`: 재시도 간격에 무작위성을 추가해서 thundering herd 방지
- `ResultPath`: 오류 정보를 입력 JSON의 특정 경로에 병합해서 Catch 상태로 전달

---

## 인간 승인(Human Approval) 패턴

Step Functions는 외부 이벤트를 기다리는 **Wait for a Callback** 패턴을 지원한다. 이메일 링크 클릭, Slack 버튼 등 사람의 입력이 필요한 워크플로우에 사용한다.

```
1. Step Functions가 Lambda 실행 시 taskToken 전달
2. Lambda가 이메일/Slack 등으로 사람에게 taskToken이 포함된 링크 발송
3. 사람이 승인 → API Gateway/Lambda가 SendTaskSuccess(taskToken) 호출
4. Step Functions 실행 재개
```

```json
"WaitForHumanApproval": {
  "Type": "Task",
  "Resource": "arn:aws:states:::lambda:invoke.waitForTaskToken",
  "Parameters": {
    "FunctionName": "SendApprovalEmail",
    "Payload": {
      "taskToken.$": "$$.Task.Token",
      "orderId.$": "$.orderId"
    }
  },
  "HeartbeatSeconds": 86400,
  "Next": "ProcessApprovedOrder"
}
```

`$$.Task.Token`에서 `$$`는 Step Functions의 컨텍스트 객체를 참조하는 표현식이다.

---

## 실제 사용 사례

### 이커머스 주문 처리 파이프라인

```
주문 접수
  → 결제 검증 (Lambda)
    → [성공] 재고 확인 (DynamoDB)
      → [재고 있음] 배송 준비 (ECS Task) + 이메일 발송 (SNS) [병렬]
        → 배송 완료 대기 (Wait)
          → 완료 알림
    → [실패] 환불 처리 → 고객 알림
```

### ML 데이터 처리 파이프라인

```
S3 파일 업로드 이벤트
  → 데이터 전처리 (Lambda)
    → 학습 작업 실행 (SageMaker)
      → 모델 평가 (Lambda)
        → [성능 기준 통과] 모델 배포 (SageMaker Endpoint)
        → [미통과] 알림 발송
```

---

## CDK로 Step Functions 정의하기

AWS CDK를 사용하면 코드로 State Machine을 정의할 수 있다.

```typescript
import * as sfn from 'aws-cdk-lib/aws-stepfunctions';
import * as tasks from 'aws-cdk-lib/aws-stepfunctions-tasks';

const processOrder = new tasks.LambdaInvoke(this, 'ProcessOrder', {
  lambdaFunction: processOrderFn,
  outputPath: '$.Payload',
});

const notifyCustomer = new tasks.SnsPublish(this, 'NotifyCustomer', {
  topic: orderTopic,
  message: sfn.TaskInput.fromJsonPathAt('$.message'),
});

const handleError = new sfn.Pass(this, 'HandleError');

const definition = processOrder
  .addCatch(handleError, { errors: ['States.ALL'] })
  .next(notifyCustomer);

new sfn.StateMachine(this, 'OrderStateMachine', {
  definition,
  timeout: Duration.minutes(5),
  stateMachineType: sfn.StateMachineType.EXPRESS,
});
```

---

## Step Functions vs 직접 코드로 오케스트레이션

| 항목 | Step Functions | 직접 코드 (Lambda → Lambda 호출) |
|------|---------------|----------------------------------|
| 실행 가시성 | 실행 그래프, 각 단계 입출력 확인 가능 | CloudWatch 로그 분석 필요 |
| 재시도/오류 처리 | 선언적 정의 | 코드로 직접 구현 |
| 장기 실행 | 1년까지 지원 | Lambda 최대 15분 제한 |
| 비용 | 상태 전환당 과금 | Lambda 실행 시간당 과금 |
| 복잡도 | 워크플로우가 복잡할수록 유리 | 단순한 체이닝은 코드가 더 간단 |

복잡한 분기, 재시도, 병렬 처리, 장기 실행이 필요한 비즈니스 로직이라면 Step Functions가 훨씬 유리하다. 반면 단순한 Lambda 두세 개를 이어붙이는 정도라면 굳이 Step Functions를 도입할 필요는 없다.

---

## 요약

- Step Functions는 여러 AWS 서비스를 조합한 워크플로우를 시각적으로 정의하고 실행하는 오케스트레이션 서비스
- State 유형: Task, Choice, Wait, Parallel, Map, Pass, Succeed, Fail
- Standard(장기, Exactly-once) vs Express(고처리량, At-least-once) 두 가지 실행 유형
- `Retry`와 `Catch`로 오류 처리를 선언적으로 구성
- `waitForTaskToken` 패턴으로 사람의 승인이 필요한 워크플로우 구현 가능
- AWS CDK, CloudFormation, Terraform으로 IaC 관리 가능
