---
layout: post
title: "[Daily morning study] Terraform을 사용한 IaC (Infrastructure as Code)"
description: >
  #daily morning study
category: 
    - dms
    - -devops
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---
# Terraform을 사용한 IaC (Infrastructure as Code)

Terraform은 인프라를 코드로 관리하는 강력한 도구입니다. Provider와 Resource를 활용하여 클라우드 인프라를 정의하고 관리하는 방법을 익힙니다. 이 학습 가이드에서는 Terraform의 기본 개념, 주요 구성 요소, 그리고 실제 예제를 통해 IaC의 원리를 설명합니다.

## 1. Terraform의 기본 개념

### Infrastructure as Code (IaC)

IaC는 인프라를 코드로 정의하여 필요한 환경을 자동으로 프로비저닝하고 관리하는 접근 방식입니다. 이는 수동 설정에서 발생할 수 있는 오류를 줄이고, 설정의 일관성을 유지하는 데 큰 도움을 줍니다.

### Terraform의 장점

- **플랫폼 독립성**: AWS, GCP, Azure 등 다양한 클라우드 플랫폼을 지원
- **모듈화**: 재사용 가능한 모듈로 관리 가능
- **버전 관리**: 코드로 관리되므로 Git 등으로 버전 관리 가능
- **계획 및 실행**: `terraform plan`과 `terraform apply` 명령어로 변화를 미리 예측하고 적용 가능

## 2. Terraform의 주요 구성 요소

### 2.1 Provider

Provider는 Terraform이 소통할 클라우드 서비스(또는 API)에 대한 인터페이스입니다. AWS, Azure, Google Cloud 등 여러 프로바이더를 사용할 수 있습니다. 예를 들어, AWS 프로바이더를 사용하기 위해서는 다음과 같이 설정합니다.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 3.0"
    }
  }
}

provider "aws" {
  region = "us-west-1"
}
```

### 2.2 Resource

Resource는 프로바이더를 통해 관리할 실제 인프라의 구성 요소입니다. 예를 들어, EC2 인스턴스를 생성하려면 다음과 같이 정의합니다.

```hcl
resource "aws_instance" "example" {
  ami           = "ami-0c55b159cbfafe1fe"
  instance_type = "t2.micro"

  tags = {
    Name = "example-instance"
  }
}
```

### 2.3 Variables

사용자가 입력한 값을 변수로 저장할 수 있습니다. 변수를 정의하려면 다음과 같이 `variables.tf` 파일을 사용할 수 있습니다.

```hcl
variable "instance_type" {
  description = "The type of instance to create"
  default     = "t2.micro"
}
```

변수를 사용하는 방법은 아래와 같습니다.

```hcl
resource "aws_instance" "example" {
  ami           = "ami-0c55b159cbfafe1fe"
  instance_type = var.instance_type

  tags = {
    Name = "example-instance"
  }
}
```

## 3. Terraform 기본 Workflow

Terraform을 사용하여 인프라를 관리하려면 대개 다음과 같은 순서를 따릅니다.

1. **구성 파일 작성**: 원하는 인프라 구성을 `.tf` 파일로 작성합니다.
2. **초기화**: `terraform init` 명령어로 필요한 프로바이더와 모듈을 초기화합니다.
3. **계획 생성**: `terraform plan` 명령어를 사용해 실제로 적용될 변경사항을 확인합니다.
4. **적용**: `terraform apply` 명령어로 인프라를 실제로 구성하고 적용합니다.
5. **상태 확인**: `terraform state` 명령어로 현재 인프라의 상태를 확인합니다.

## 4. 실습 예제

아래의 예제는 AWS에서 EC2 인스턴스를 생성하는 간단한 Terraform 코드입니다.

### 4.1 `main.tf` 파일 생성

```hcl
provider "aws" {
  region = "us-west-1"
}

resource "aws_instance" "example" {
  ami           = "ami-0c55b159cbfafe1fe"
  instance_type = "t2.micro"

  tags = {
    Name = "example-instance"
  }
}
```

### 4.2 Terraform 초기화 및 실행

```bash
terraform init
terraform plan
terraform apply
```

## 5. 기타 유용한 명령어

- `terraform destroy`: Terraform으로 만든 리소스를 삭제합니다.
- `terraform output`: 출력값을 조회합니다.
- `terraform fmt`: 코드 포맷을 일관되게 정리합니다.

## 6. 마무리

Terraform을 사용한 IaC는 인프라를 코드로 관리하는 효율적이고 유용한 방법입니다. 기본 개념과 예제를 통해 Terraform을 활용해 원하는 인프라를 손쉽게 구축하고 관리해보세요. 이 가이드는 Terraform을 시작하는 데 도움이 되는 기본적인 정보를 제공합니다. 더 심화된 내용은 공식 문서나 추가 자료를 통해 학습할 수 있습니다.
