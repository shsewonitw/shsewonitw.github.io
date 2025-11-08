---
layout: post
title: "[Daily morning study] AWS IAM (Identity and Access Management)의 중요성"
description: >
  #daily morning study
category: 
    - dms
    - -cloud
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---
# AWS IAM (Identity and Access Management)의 중요성

AWS IAM은 Amazon Web Services에서 제공하는 신원 및 접근 관리 서비스입니다. IAM을 통해 사용자는 AWS 리소스에 대한 접근을 제어할 수 있으며, 보안과 관리 측면에서 매우 중요합니다. 

## IAM의 기본 개념

### 사용자(User)
- IAM 사용자는 AWS 계정에 속한 individuall 사용자를 의미합니다. 각 사용자는 고유한 자격 증명을 통해 시스템에 접근할 수 있습니다.

### 그룹(Group)
- IAM 그룹은 여러 사용자들을 하나로 묶어 공통된 정책을 적용할 수 있게 해줍니다. 새로운 사용자에게 그룹을 통해 권한을 부여하는 것이 효율적입니다.

### 역할(Role)
- IAM 역할은 필요한 권한을 가진 사용자나 서비스를 대신하여 AWS 리소스에 접근할 수 있게 해줍니다. 이는 주로 AWS 서비스가 다른 서비스를 호출할 때 사용됩니다.

### 정책(Policy)
- IAM 정책은 JSON 형식으로 작성하며, 특정 권한을 부여하는 심플한 규칙 집합입니다. 예를 들면 EC2 인스턴스에 대한 생성, 수정, 삭제 권한을 설정할 수 있습니다.

## 왜 IAM이 중요한가?

### 1. 리소스 보호
IAM을 통해 AWS 리소스에 대한 접근을 강력하게 제어할 수 있습니다. 이를 통해 민감한 정보를 보호하고 데이터 유출을 최소화할 수 있습니다.

### 2. 최소 권한 원칙
IAM은 최소 권한 원칙을 적용하여 사용자가 작업을 수행하는 데 필요한 최소한의 권한만 부여할 수 있도록 합니다. 이를 통해 위험 요소를 줄일 수 있습니다.

### 3. 감사 및 모니터링
IAM은 AWS CloudTrail과 통합되어 보안 감사 기록을 생성합니다. 어떤 사용자가 누구에게 어떤 접근 권한을 요청했는지, 어떤 작업을 수행했는지를 트래킹할 수 있습니다.

### 4. 다양한 인증 옵션
IAM은 여러 인증 방법을 지원합니다. Multi-Factor Authentication(MFA)과 같은 방법을 통해 추가적인 보안을 강화할 수 있습니다.

## IAM 기본 활용 예제

### 사용자 생성 및 정책 부여

다음은 IAM에서 사용자를 생성하고 정책을 부여하는 기본적인 예제입니다. AWS CLI를 사용하여 진행할 수 있습니다.

```bash
# 사용자 생성
aws iam create-user --user-name new-user

# 정책 생성 예 (S3 접근 권한 부여)
aws iam create-policy --policy-name S3AccessPolicy --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "s3:*",
            "Resource": "*"
        }
    ]
}'

# 사용자에게 정책 연결
aws iam attach-user-policy --user-name new-user --policy-arn arn:aws:iam::account-id:policy/S3AccessPolicy
```

## IAM의 베스트 프랙티스

1. **정기적인 접근 권한 검토**: 사용자의 권한을 주기적으로 점검하고 불필요한 권한을 정리합니다.
2. **MFA 활성화**: 중요한 사용자에게는 반드시 MFA를 설정합니다.
3. **타겟 사용자 그룹화**: 비슷한 역할을 지닌 사용자들을 그룹으로 모아 정책을 간편하게 관리합니다.
4. **정책은 최소 권한 원칙을 따르기**: 사용자가 실제로 필요한 권한만 부여합니다.
5. **자동화 도구 활용**: AWS Config, CloudTrail 등을 사용하여 권한 변경 및 활동을 모니터링하고 기록합니다.

## 결론

IAM은 AWS 환경 내에서 보안을 유지하고, 리소스에 대한 접근을 통제하는 도구로서 매우 중요한 역할을 합니다. IAM을 제대로 이해하고 운용하는 것은 AWS를 안전하게 사용하는 기본적인 요소입니다. 각 기능과 베스트 프랙티스를 숙지하여 효율적인 보안 관리를 수행하길 바랍니다.
