---
layout: post
title: "[Daily morning study] Serverless 아키텍처와 AWS Lambda (또는 Google Cloud Functions)"
description: >
  #daily morning study
category: 
    - dms
    - -backend
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---
# Serverless 아키텍처와 AWS Lambda / Google Cloud Functions

서버리스 아키텍처는 클라우드 서비스 제공업체가 서버 관리의 대부분을 처리하여 개발자가 비즈니스 로직에 더 집중할 수 있게 해주는 방식을 의미한다. 여기서는 AWS Lambda와 Google Cloud Functions를 중심으로 서버리스 아키텍처의 개념과 이점, 사용 시나리오에 대해 알아볼 것이다.

## 1. 서버리스 아키텍처란?

서버리스 아키텍처는 사용자가 서버를 직접 프로비저닝 및 관리하지 않도록 해준다. 대신, 클라우드 서비스 제공업체가 인프라를 관리한다. 개발자는 애플리케이션 코드와 비즈니스 로직에만 중점을 두면 된다.

### 장점:

- **자동 확장**: 수요에 따라 자동으로 리소스를 할당해준다.
- **비용 효율성**: 사용한 만큼 비용을 지불하고, 서버 유지 관리 비용을 절감할 수 있다.
- **빠른 개발**: 복잡한 인프라 설정이 필요 없기 때문에 빠른 배포가 가능하다.

## 2. AWS Lambda

AWS Lambda는 아마존 웹 서비스에서 제공하는 서버리스 컴퓨팅 플랫폼이다. 코드를 업로드하면 Lambda가 자동으로 실행되고, 필요할 때만 비용을 지불하면 된다.

### 주요 특징:

- **이벤트 기반**: S3, DynamoDB, API Gateway와 같은 다양한 AWS 서비스에서 이벤트를 통해 트리거된다.
- **언어 지원**: Java, Python, Node.js, C# 등 여러 언어를 지원한다.
- **컨테이너 지원**: Docker 컨테이너를 이용한 배포도 지원된다.

### 코드 예제 (AWS Lambda)

```python
import json

def lambda_handler(event, context):
    # 이벤트에서 값 가져오기
    name = event.get("name", "World")
    return {
        'statusCode': 200,
        'body': json.dumps(f"Hello, {name}!")
    }
```

## 3. Google Cloud Functions

Google Cloud Functions는 구글 클라우드에서 제공하는 서버리스 환경이다. 간단한 코드 스니펫으로 다양한 클라우드 기반 서비스를 구축할 수 있다.

### 주요 특징:

- **자동 확장**: 필요에 따라 자동으로 인스턴스가 생성되고 제거된다.
- **RESTful API**: HTTP 요청에 응답하는 웹 후크를 쉽게 만들 수 있다.
- **다양한 언어 지원**: Node.js, Python, Go, Java 등 다양한 언어를 지원한다.

### 코드 예제 (Google Cloud Functions)

```python
def hello_world(request):
    request_json = request.get_json()
    name = request_json.get("name", "World")
    
    return f"Hello, {name}!"
```

## 4. 사용 시나리오

서버리스 아키텍처는 여러 상황에서 유용하게 사용될 수 있다.

### 웹 애플리케이션

API 백엔드를 서버리스로 구현하여 요청마다 필요한 리소스만 사용하게 할 수 있다.

### 데이터 처리

파일 업로드 시 Lambda 또는 Cloud Functions를 사용하여 데이터를 처리하고 변환할 수 있다.

### IoT 솔루션

IoT 기기의 데이터를 수집하고 분석하기 위해 이벤트 처리기로서 서버리스를 활용할 수 있다.

## 5. 한계와 고려사항

서버리스 아키텍처는 분명 많은 이점을 갖고 있지만, 몇 가지 한계도 존재한다.

- **콜드 스타트**: 처음 호출 시 지연이 발생할 수 있다.
- **제한된 실행 시간**: Lambda는 최대 15분, Cloud Functions는 9분으로 실행 시간이 제한된다.
- **복잡한 배포 관리**: 여러 개의 함수와 서비스를 관리하게 되어 복잡해질 수 있다.

## 6. 결론

서버리스 아키텍처는 현대 웹 애플리케이션 개발에 있어 강력한 옵션이 될 수 있다. AWS Lambda와 Google Cloud Functions 모두 쉽게 사용 가능한 플랫폼이므로, 각자의 필요에 맞게 선택하여 활용할 수 있다. 필요한 로직을 작성하고 클라우드에 배포하는 과정에서 인프라 관리를 단순화하고, 비즈니스 로직에 더욱 집중할 수 있는 환경을 제공한다.
