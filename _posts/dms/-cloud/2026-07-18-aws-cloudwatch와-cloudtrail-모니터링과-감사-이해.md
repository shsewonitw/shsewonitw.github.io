---
layout: post
title: "[Daily morning study] AWS CloudWatch와 CloudTrail — 모니터링과 감사(Audit) 이해"
description: >
  #daily morning study
category: 
    - dms
    - dms-cloud
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## CloudWatch와 CloudTrail, 뭐가 다른가?

AWS를 쓰다 보면 CloudWatch와 CloudTrail을 헷갈리는 경우가 많다. 이름도 비슷하고 둘 다 "무언가를 기록한다"는 느낌이라서 그런데, 목적이 완전히 다르다.

| 구분 | CloudWatch | CloudTrail |
|------|-----------|------------|
| 목적 | **성능 모니터링** (Metrics, Logs, Alarms) | **API 활동 감사** (누가 무엇을 했는가) |
| 대상 | EC2 CPU, Lambda 실행 시간, 에러 로그 등 | AWS 콘솔, CLI, SDK 호출 이력 |
| 저장 단위 | 메트릭(수치), 로그 스트림 | 이벤트 레코드 (JSON) |
| 주요 사용처 | 알람 설정, 대시보드, 로그 분석 | 보안 감사, 규정 준수, 포렌식 |

---

## AWS CloudWatch

CloudWatch는 AWS 리소스와 애플리케이션의 **운영 데이터를 수집·시각화·알림**하는 서비스다.

### 핵심 구성 요소

**Metrics (메트릭)**

AWS 서비스에서 자동으로 수집되는 시계열 수치 데이터다.

- EC2: `CPUUtilization`, `NetworkIn`, `DiskReadBytes`
- RDS: `DatabaseConnections`, `FreeStorageSpace`
- Lambda: `Invocations`, `Errors`, `Duration`

기본 메트릭은 5분 간격으로 수집되고, 세부 모니터링(Detailed Monitoring)을 활성화하면 1분 단위가 된다. 커스텀 메트릭은 `PutMetricData` API로 직접 올릴 수 있다.

```bash
# 커스텀 메트릭 전송 예시
aws cloudwatch put-metric-data \
  --namespace "MyApp" \
  --metric-name "ActiveUsers" \
  --value 423 \
  --unit Count
```

**Logs (로그)**

애플리케이션, EC2 시스템 로그, Lambda 실행 로그 등을 중앙에서 수집한다. 구조는 다음과 같다.

```
Log Group  ─┬─ /aws/lambda/my-function
             └─ /var/log/nginx/access.log

Log Stream ─┬─ 2026/07/18/[$LATEST]abc123
             └─ 2026/07/18/[$LATEST]def456
```

- **Log Group**: 동일한 소스의 로그를 묶는 단위
- **Log Stream**: 인스턴스 하나 또는 실행 하나의 로그 흐름

**CloudWatch Logs Insights**를 쓰면 SQL과 유사한 쿼리로 로그를 분석할 수 있다.

```
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 20
```

**Alarms (알람)**

메트릭이 특정 임계값을 넘으면 알림을 보내거나 자동 액션을 실행한다.

```
EC2 CPUUtilization > 80% (5분간 지속)
  → SNS 알림 발송
  → Auto Scaling 스케일 아웃 트리거
```

알람 상태는 세 가지다.

- `OK`: 정상 범위
- `ALARM`: 임계값 초과
- `INSUFFICIENT_DATA`: 데이터 부족 (초기 상태 또는 메트릭 없음)

**Dashboards**

여러 메트릭을 하나의 화면에 모아서 볼 수 있는 커스텀 대시보드다. 팀마다 CPU, 메모리, 에러율, 요청 수를 한눈에 볼 수 있는 운영 화면을 구성할 때 사용한다.

---

## AWS CloudTrail

CloudTrail은 AWS 계정 내 **모든 API 호출을 기록**하는 감사 로그 서비스다. "누가, 언제, 어디서, 무엇을 했는가"를 추적한다.

### 기록되는 정보

CloudTrail이 캡처하는 이벤트 하나에는 다음 정보가 담긴다.

```json
{
  "eventTime": "2026-07-18T09:12:34Z",
  "eventName": "DeleteSecurityGroup",
  "eventSource": "ec2.amazonaws.com",
  "userIdentity": {
    "type": "IAMUser",
    "userName": "alice",
    "accountId": "123456789012"
  },
  "sourceIPAddress": "203.0.113.10",
  "requestParameters": {
    "groupId": "sg-0a1b2c3d4e5f"
  },
  "responseElements": null
}
```

- 어떤 API를 호출했는지 (`DeleteSecurityGroup`)
- 누가 호출했는지 (`alice`)
- 어디서 호출했는지 (`sourceIPAddress`)
- 성공/실패 여부

### 이벤트 종류

| 유형 | 설명 | 예시 |
|------|------|------|
| Management Events | 제어 플레인 작업 (기본 기록) | EC2 생성/삭제, IAM 정책 변경 |
| Data Events | 데이터 플레인 작업 (별도 설정 필요) | S3 오브젝트 읽기/쓰기, Lambda 실행 |
| Insights Events | 비정상적인 API 활동 패턴 감지 | 갑작스러운 `TerminateInstances` 급증 |

Data Events는 볼륨이 매우 크기 때문에 기본으로 수집하지 않고, 필요한 S3 버킷이나 Lambda 함수만 선택적으로 활성화한다.

### Trail 설정

Trail을 만들면 이벤트 로그가 S3 버킷에 자동 저장된다. CloudWatch Logs와 연결하면 실시간 이벤트 스트리밍도 가능하다.

```
CloudTrail Trail
  ├── S3 버킷 (장기 보관, 압축 JSON)
  └── CloudWatch Logs (실시간 분석, 알람 연동)
```

모든 리전의 이벤트를 하나의 Trail로 수집하려면 **Multi-Region Trail**을 활성화해야 한다. 기본적으로 Trail은 특정 리전에만 적용된다.

---

## 두 서비스를 함께 쓰는 패턴

실제 운영 환경에서는 CloudWatch와 CloudTrail을 조합해서 쓴다.

**시나리오 1: 보안 이벤트 실시간 알람**

CloudTrail → CloudWatch Logs → Metric Filter → Alarm → SNS

```
# 루트 계정 로그인 발생 시 알람
CloudTrail: RootLogin 이벤트 감지
  → CloudWatch Logs에 스트리밍
  → Metric Filter로 패턴 매칭
  → 알람 발생
  → SNS → 슬랙/이메일 알림
```

**시나리오 2: 장애 원인 분석**

```
1. CloudWatch Alarm: Lambda 에러율 급증 감지
2. CloudWatch Logs: 에러 스택 트레이스 확인
3. CloudTrail: 에러 직전 배포/설정 변경 이력 조회
```

---

## 요금 구조 요점

**CloudWatch**

- 기본 메트릭(5분): 무료
- 커스텀 메트릭: 메트릭 개수 × 월 단위 과금
- Logs 수집: GB당 수집 요금 + 저장 요금
- Alarms: 알람 개수당 과금

**CloudTrail**

- Management Events: 계정당 첫 Trail은 무료 (S3 저장 비용만 발생)
- Data Events: 이벤트 100,000건당 과금
- Insights: 분석 이벤트 수에 따라 과금

---

## 정리

- CloudWatch: **"지금 시스템이 잘 돌아가고 있나?"** → 메트릭, 로그, 알람
- CloudTrail: **"누가 뭘 바꿨나?"** → API 호출 이력, 보안 감사

실무에서 두 서비스를 같이 설정해두면 장애 대응 속도가 크게 달라진다. CloudWatch로 이상을 빠르게 감지하고, CloudTrail로 원인을 역추적하는 흐름이 기본이다.
