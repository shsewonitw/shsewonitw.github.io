---
layout: post
title: "[Daily morning study] 시크릿 관리 (HashiCorp Vault, AWS Secrets Manager)"
description: >
  #daily morning study
category: 
    - dms
    - dms-devops
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 시크릿 관리가 왜 중요한가

애플리케이션을 운영하다 보면 DB 비밀번호, API 키, TLS 인증서, OAuth 토큰 등 민감한 정보를 어딘가에 저장해야 한다. 이런 값들을 소스 코드나 환경 변수 파일에 하드코딩하면 Git에 노출되거나 배포 과정에서 유출될 위험이 있다.

시크릿 관리 도구는 이런 민감 정보를 중앙에서 안전하게 저장하고, 접근 제어를 통해 필요한 서비스에만 필요한 순간에만 전달한다.

---

## HashiCorp Vault

### 개념

Vault는 HashiCorp이 만든 오픈소스 시크릿 관리 플랫폼이다. 자체 호스팅이 가능하고 클라우드에 종속되지 않는다는 점이 가장 큰 장점이다.

### 핵심 개념

| 개념 | 설명 |
|------|------|
| Secret Engine | 시크릿을 저장하거나 생성하는 플러그인 (KV, Database, AWS, PKI 등) |
| Auth Method | Vault에 인증하는 방법 (Token, AppRole, Kubernetes, AWS IAM 등) |
| Policy | 어떤 경로의 시크릿에 어떤 권한을 줄지 정의 |
| Lease | 동적 시크릿의 유효 기간 |

### KV (Key-Value) Secret Engine

가장 기본적인 방식으로 정적 시크릿을 저장한다.

```bash
# KV v2 활성화
vault secrets enable -path=secret kv-v2

# 시크릿 저장
vault kv put secret/myapp/db \
  username="admin" \
  password="s3cr3t"

# 시크릿 조회
vault kv get secret/myapp/db

# JSON 출력
vault kv get -format=json secret/myapp/db | jq '.data.data'
```

### 동적 시크릿 (Dynamic Secrets)

Vault의 가장 강력한 기능 중 하나다. 요청할 때마다 임시 자격증명을 생성하고, Lease가 만료되면 자동으로 삭제한다.

```bash
# Database secret engine 활성화
vault secrets enable database

# PostgreSQL 연결 설정
vault write database/config/my-postgresql \
  plugin_name=postgresql-database-plugin \
  allowed_roles="readonly" \
  connection_url="postgresql://{{username}}:{{password}}@db:5432/mydb" \
  username="vault" \
  password="vaultpass"

# Role 생성 (임시 계정 템플릿)
vault write database/roles/readonly \
  db_name=my-postgresql \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"

# 임시 자격증명 발급
vault read database/creds/readonly
```

이렇게 하면 애플리케이션은 매번 새로운 임시 DB 계정을 받아서 사용하고, 1시간 뒤에는 자동 삭제된다.

### Policy 설정

```hcl
# readonly-policy.hcl
path "secret/data/myapp/*" {
  capabilities = ["read", "list"]
}

path "database/creds/readonly" {
  capabilities = ["read"]
}
```

```bash
vault policy write readonly-policy readonly-policy.hcl
```

### Kubernetes와의 연동

Kubernetes 환경에서 Pod가 Vault에 인증하는 대표적인 방법이다.

```bash
# Kubernetes Auth Method 활성화
vault auth enable kubernetes

# K8s 클러스터 정보 등록
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  token_reviewer_jwt=@/var/run/secrets/kubernetes.io/serviceaccount/token

# Role 생성 (어떤 ServiceAccount가 어떤 Policy를 사용할지)
vault write auth/kubernetes/role/myapp \
  bound_service_account_names=myapp-sa \
  bound_service_account_namespaces=default \
  policies=readonly-policy \
  ttl=24h
```

Pod에서는 Vault Agent Injector나 CSI Driver를 통해 자동으로 시크릿을 주입받을 수 있다.

```yaml
# Vault Agent Injector 어노테이션 예시
annotations:
  vault.hashicorp.com/agent-inject: "true"
  vault.hashicorp.com/role: "myapp"
  vault.hashicorp.com/agent-inject-secret-config.env: "secret/data/myapp/db"
```

---

## AWS Secrets Manager

### 개념

AWS가 제공하는 완전 관리형 시크릿 관리 서비스다. AWS 환경에 깊게 통합되어 있고 별도 서버 운영이 필요 없다.

### 주요 특징

- 자동 로테이션 지원 (RDS, Redshift 등과 네이티브 연동)
- IAM 기반 접근 제어
- KMS를 통한 암호화
- CloudTrail로 접근 로그 감사

### 시크릿 생성 및 조회

```bash
# AWS CLI로 시크릿 생성
aws secretsmanager create-secret \
  --name myapp/db \
  --secret-string '{"username":"admin","password":"s3cr3t"}'

# 시크릿 조회
aws secretsmanager get-secret-value \
  --secret-id myapp/db \
  --query SecretString \
  --output text | jq .
```

### 자동 로테이션

```bash
# RDS 자격증명 자동 로테이션 활성화 (30일마다)
aws secretsmanager rotate-secret \
  --secret-id myapp/db \
  --rotation-rules AutomaticallyAfterDays=30
```

AWS가 Lambda 함수를 자동 생성하여 주기적으로 비밀번호를 교체하고 RDS에도 반영한다.

### 애플리케이션에서 사용 (Python)

```python
import boto3
import json

def get_secret(secret_name: str) -> dict:
    client = boto3.client("secretsmanager", region_name="ap-northeast-2")
    response = client.get_secret_value(SecretId=secret_name)
    return json.loads(response["SecretString"])

# 사용
db_creds = get_secret("myapp/db")
conn = psycopg2.connect(
    host="db.example.com",
    user=db_creds["username"],
    password=db_creds["password"],
    dbname="mydb"
)
```

### IAM 정책으로 접근 제어

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:ap-northeast-2:123456789012:secret:myapp/*"
    }
  ]
}
```

EC2, ECS, Lambda에 이 정책을 가진 IAM Role을 부여하면, 코드에 자격증명 없이 시크릿에 접근할 수 있다.

---

## AWS Parameter Store와의 차이

AWS에는 Secrets Manager 외에도 **SSM Parameter Store**라는 유사한 서비스가 있다.

| 항목 | Secrets Manager | Parameter Store |
|------|----------------|----------------|
| 비용 | 시크릿당 월 $0.40 | 표준 파라미터 무료 |
| 자동 로테이션 | 지원 | 미지원 (직접 구현) |
| 버전 관리 | 지원 | 지원 |
| 주 용도 | DB 자격증명, API 키 등 민감 정보 | 설정값, 환경변수 등 |
| 크기 제한 | 64KB | 표준 4KB / 어드밴스드 8KB |

민감도가 높고 자동 로테이션이 필요하면 Secrets Manager, 단순 설정값이면 Parameter Store를 쓰는 게 일반적이다.

---

## Vault vs AWS Secrets Manager 비교

| 항목 | HashiCorp Vault | AWS Secrets Manager |
|------|----------------|---------------------|
| 호스팅 | 자체 호스팅 (또는 HCP Vault) | 완전 관리형 |
| 클라우드 종속성 | 없음 (멀티 클라우드 가능) | AWS 종속 |
| 동적 시크릿 | 지원 (DB, Cloud 자격증명 동적 생성) | 제한적 (RDS 로테이션 정도) |
| 인증 방법 | Token, AppRole, LDAP, K8s, AWS 등 다양 | IAM 기반 |
| 학습 곡선 | 높음 | 낮음 (AWS에 익숙하다면) |
| 비용 | 오픈소스 무료 (운영 비용 발생) | 사용량 기반 과금 |

AWS 중심으로 운영한다면 Secrets Manager가 간편하고, 멀티 클라우드나 온프레미스를 함께 쓴다면 Vault가 적합하다.

---

## 실무에서의 베스트 프랙티스

1. **코드에 시크릿을 절대 하드코딩하지 않는다.** `.gitignore`로 막는 것만으로는 부족하다.
2. **최소 권한 원칙**: 각 서비스는 자신에게 필요한 시크릿에만 접근 가능하도록 Policy를 좁게 설정한다.
3. **로테이션 주기를 짧게**: 유출되더라도 피해 기간을 최소화한다.
4. **접근 로그 감사**: 누가 언제 어떤 시크릿에 접근했는지 추적한다.
5. **환경별 시크릿 분리**: `dev/myapp/db`, `prod/myapp/db` 처럼 경로로 환경을 구분한다.
