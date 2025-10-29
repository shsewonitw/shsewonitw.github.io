---
layout: post
title: "[Daily morning study] Ansible의 역할과 Terraform과의 차이 (Configuration Management vs. Provisioning)"
description: >
  #daily morning study
category: 
    - dms
    - -devops
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---
# Ansible의 역할과 Terraform과의 차이 (Configuration Management vs. Provisioning)

Ansible과 Terraform은 오늘날 IT 인프라 관리에서 자주 사용되는 두 가지 도구입니다. 이 문서에서는 이 두 도구의 역할, 기능, 그리고 서로 다른 점을 살펴보겠습니다.

## Ansible 

### Ansible 개요
Ansible은 간단하고 강력한 구성 관리 도구입니다. 애플리케이션 배포, 시스템 구성 및 다양한 자동화 작업에 사용됩니다. Ansible은 에이전트가 필요 없는 방식으로 작동하며, SSH(안전한 쉘 프로토콜)를 통해 관리 대상 시스템에 연결합니다.

### 주요 기능
- **구성 관리:** 시스템의 설정을 정의하고 일관된 상태를 유지하도록 관리합니다.
- **애플리케이션 배포:** 소프트웨어 패키지를 여러 서버에 간단하게 배포할 수 있습니다.
- **클라우드 통합:** AWS, Azure 등 클라우드 환경에서도 쉽게 사용할 수 있습니다.

### 기본 구성 요소
- **Playbook:** Ansible의 가장 중요한 구성 요소로, YAML 형식으로 작성된 스크립트입니다. 여러 작업을 정의하고 orchestrate합니다.
- **Inventory:** 관리할 서버의 목록을 정의합니다. IP 주소 또는 호스트 이름을 포함합니다.
- **모듈:** 특정 작업을 수행하기 위한 코드 조각으로, 시스템에 직접적으로 명령을 전달합니다.

### 간단한 Example
아래는 웹 서버를 설치하는 Ansible Playbook의 예시입니다.

```yaml
- name: Install and configure web server
  hosts: webservers
  become: yes
  tasks:
    - name: Install Apache
      yum:
        name: httpd
        state: present

    - name: Start Apache
      service:
        name: httpd
        state: started
```

## Terraform 

### Terraform 개요
Terraform은 인프라를 코드(Infrastructure as Code)로 정의하고 관리하는 도구입니다. 인프라 자원(서버, 데이터베이스 등)을 쉽게 프로비저닝하고 관리할 수 있게 해줍니다.

### 주요 기능
- **프로비저닝:** 클라우드 환경에서의 모든 자원을 자동으로 생성하고 관리합니다.
- **상태 관리:** 모든 인프라 자원의 현재 상태를 기록합니다. 이를 통해 변경 사항을 쉽게 추적할 수 있습니다.
- **모듈화:** 코드 재사용성을 높이기 위해 모듈화된 구성을 지원합니다.

### 기본 구성 요소
- **Configuration Files:** Terraform의 설정 파일은 HCL(HashiCorp Configuration Language) 형식으로 작성됩니다.
- **Providers:** 다양한 클라우드 서비스 제공업체와 연동하여 리소스를 생성하고 관리합니다.
- **State File:** 현재의 인프라 상태를 저장하는 파일로, Terraform이 변경 사항을 추적하는 데 사용됩니다.

### 간단한 Example
아래는 AWS에서 EC2 인스턴스를 생성하는 Terraform 코드 예시입니다.

```hcl
provider "aws" {
  region = "us-west-2"
}

resource "aws_instance" "example" {
  ami           = "ami-0c55b159cbfafe1fe"
  instance_type = "t2.micro"

  tags = {
    Name = "MyExampleInstance"
  }
}
```

## Ansible vs Terraform

| 특성               | Ansible                             | Terraform                          |
|------------------|-----------------------------------|-----------------------------------|
| 목적               | 구성 관리 (Configuration Management)  | 인프라 프로비저닝 (Provisioning)  |
| 작동 방식          | 에이전트가 필요 없는 SSH 기반         | 상태 파일 기반                     |
| 언어               | YAML                              | HCL (HashiCorp Configuration Language) |
| 상태 관리          | 무상태 (Stateless)                 | 상태 관리 (State Management)        |
| 배포주기          | 주로 사용자의 요청에 따라                   | 선언적으로 인프라를 정의하고 변경 사항을 관리  |

## 결론 

Ansible과 Terraform은 각각 다른 목적을 가지고 있지만, 둘 다 자동화와 관리를 통해 인프라 효율성을 높여줍니다. Ansible은 구성 관리에 초점을 두고 있으며, Terraform은 인프라의 프로비저닝 및 상태 관리를 담당합니다. 따라서 필요에 따라 그 사용 목적에 맞게 선택하는 것이 중요합니다. Ansible은 일관된 구성 관리에, Terraform은 자원 생성과 변경 관리에 적합합니다.
