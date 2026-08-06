---
layout: post
title: "[Daily morning study] AWS API Gateway 개념과 활용"
description: >
  #daily morning study
category: 
    - dms
    - dms-cloud
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## AWS API Gateway란

AWS API Gateway는 REST API, HTTP API, WebSocket API를 생성·배포·관리하는 완전 관리형 서비스다. 클라이언트 요청을 받아 백엔드(Lambda, EC2, ECS, 외부 HTTP 엔드포인트 등)로 라우팅하고, 인증·캐싱·속도 제한·모니터링을 한곳에서 처리한다.

서버를 직접 운영하지 않아도 되기 때문에 Serverless 아키텍처에서 Lambda와 함께 쓰는 조합이 가장 일반적이다.

---

## API Gateway가 제공하는 세 가지 API 유형

| 유형 | 프로토콜 | 특징 | 주요 사용 사례 |
|------|---------|------|--------------|
| REST API | HTTP/HTTPS | 기능이 가장 풍부, 캐싱·Usage Plan·요청 검증 등 지원 | 레거시 연동, 정교한 API 관리 필요 시 |
| HTTP API | HTTP/HTTPS | REST API보다 단순하고 저렴(약 70% 저렴), 지연 시간도 낮음 | 간단한 Lambda 프록시, OIDC·OAuth 2.0 인증 |
| WebSocket API | WebSocket | 클라이언트-서버 양방향 실시간 통신 지원 | 채팅, 알림 푸시, 실시간 게임 |

새 프로젝트라면 대부분 **HTTP API**로 시작하는 게 낫다. REST API는 캐싱이나 API Key 관리 같은 고급 기능이 실제로 필요할 때 선택한다.

---

## 핵심 구성 요소

### 리소스(Resource)와 메서드(Method)

REST API 기준으로 API Gateway는 URL 경로를 **리소스**로, HTTP 동사를 **메서드**로 관리한다.

```
/users                  ← 리소스
  GET   → Lambda A      ← 메서드 + 통합
  POST  → Lambda B

/users/{userId}         ← 경로 파라미터를 포함한 리소스
  GET   → Lambda C
  DELETE → Lambda D
```

### 통합(Integration) 유형

| 통합 유형 | 설명 |
|----------|------|
| Lambda 프록시 통합 | 요청 전체를 Lambda에 그대로 전달, Lambda가 응답 객체를 직접 반환 |
| Lambda 비프록시 통합 | 매핑 템플릿으로 요청/응답 변환 가능, 세밀한 제어 |
| HTTP 통합 | 외부 HTTP 엔드포인트로 프록시 |
| AWS 서비스 통합 | DynamoDB, SQS, Step Functions 등 AWS 서비스 직접 호출 |
| Mock 통합 | 백엔드 없이 고정 응답 반환, 개발/테스트용 |

Lambda 프록시 통합이 가장 많이 쓰인다. Lambda 함수가 `statusCode`, `headers`, `body`를 포함한 객체를 반환하면 API Gateway가 그대로 클라이언트에 전달한다.

```javascript
// Lambda 프록시 통합 응답 형식
exports.handler = async (event) => {
  const userId = event.pathParameters.userId;
  const user = await getUserById(userId);

  return {
    statusCode: 200,
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*'
    },
    body: JSON.stringify(user)
  };
};
```

### 스테이지(Stage)

API Gateway는 배포 단위로 **스테이지**를 사용한다. `dev`, `staging`, `prod` 같은 환경을 분리해서 운영할 수 있다.

```
https://{api-id}.execute-api.{region}.amazonaws.com/{stage}/users

# 예시
https://abc123.execute-api.ap-northeast-2.amazonaws.com/prod/users
https://abc123.execute-api.ap-northeast-2.amazonaws.com/dev/users
```

스테이지 변수(Stage Variables)를 사용하면 같은 API 정의로 환경별로 다른 Lambda 함수나 엔드포인트를 가리킬 수 있다.

---

## 인증·인가 방식

### 1. IAM 권한

AWS 자격증명(SigV4 서명)으로 요청을 인증한다. AWS 서비스 간 내부 통신에 적합하다.

### 2. Amazon Cognito 사용자 풀

Cognito에서 발급한 JWT 토큰을 Authorization 헤더에 담아 요청한다. 웹·모바일 앱의 사용자 인증에 많이 쓰인다.

```
Authorization: Bearer {Cognito JWT 토큰}
```

API Gateway가 토큰을 Cognito에 검증 요청하고, 유효하면 백엔드로 요청을 전달한다.

### 3. Lambda Authorizer (커스텀 인증)

토큰 검증 로직을 Lambda 함수로 직접 구현한다. 서드파티 OAuth 공급자, 레거시 인증 시스템과 연동할 때 유용하다.

```javascript
// Lambda Authorizer 예시
exports.handler = async (event) => {
  const token = event.authorizationToken;

  // 토큰 검증 로직
  const decoded = verifyToken(token);
  if (!decoded) {
    throw new Error('Unauthorized');
  }

  // IAM 정책 반환
  return {
    principalId: decoded.userId,
    policyDocument: {
      Version: '2012-10-17',
      Statement: [{
        Action: 'execute-api:Invoke',
        Effect: 'Allow',
        Resource: event.methodArn
      }]
    },
    context: {
      userId: decoded.userId,
      role: decoded.role
    }
  };
};
```

Lambda Authorizer는 결과를 최대 3600초 캐싱할 수 있어 매 요청마다 검증 Lambda가 호출되는 오버헤드를 줄인다.

### 4. API Key

API Key를 발급하고 Usage Plan에 연결해 외부 개발자에게 제공한다. 서비스 간 인증보다는 사용량 제어가 주목적이다.

```
x-api-key: {발급된 API Key}
```

---

## 요청/응답 처리 기능

### 요청 검증

백엔드를 호출하기 전에 API Gateway 레벨에서 파라미터와 요청 본문의 형식을 검증할 수 있다. 잘못된 요청을 Lambda가 받기 전에 차단해서 불필요한 Lambda 호출을 막는다.

```json
// 요청 본문 검증용 JSON Schema 예시
{
  "type": "object",
  "required": ["name", "email"],
  "properties": {
    "name":  { "type": "string" },
    "email": { "type": "string", "format": "email" }
  }
}
```

### 매핑 템플릿 (REST API)

VTL(Velocity Template Language)을 사용해서 요청·응답 형식을 변환한다. 레거시 백엔드가 다른 JSON 구조를 요구할 때 유용하다.

```
## 요청 본문 변환 예시 (VTL)
{
  "userId": "$input.params('userId')",
  "timestamp": "$context.requestTime"
}
```

### 응답 변환 및 오류 처리

백엔드에서 반환하는 오류 메시지를 API Gateway 레벨에서 클라이언트용으로 다시 포맷할 수 있다. 내부 구조를 외부에 노출하지 않을 수 있다.

---

## 속도 제한 (Throttling)

API Gateway는 두 단계로 속도를 제한한다.

| 레벨 | 기본 한도 | 설명 |
|------|---------|------|
| 계정/리전 레벨 | 10,000 RPS, 버스트 5,000 | AWS 계정 전체 한도 |
| 스테이지/메서드 레벨 | 커스텀 설정 가능 | 특정 API나 메서드에만 별도 한도 적용 |
| Usage Plan | API Key별 설정 | 외부 개발자별 할당량 제어 |

한도를 초과하면 `429 Too Many Requests`를 반환한다. 백엔드 Lambda가 폭발적으로 호출되는 걸 막는 1차 방어선 역할을 한다.

---

## 캐싱 (REST API만 지원)

REST API 스테이지에서 응답 캐싱을 활성화하면 동일한 요청에 대해 백엔드를 호출하지 않고 캐시된 응답을 반환한다.

- 캐시 크기: 0.5 GB ~ 237 GB
- TTL: 기본 300초 (0~3600초 설정 가능)
- 메서드 단위로 캐싱 활성화/비활성화 가능
- 캐시 키에 헤더·쿼리 파라미터·경로 파라미터 포함 가능

캐싱을 사용하면 비용이 추가되고 캐시 무효화 전략을 별도로 설계해야 하므로, 실제로 반복 요청이 많고 응답이 자주 변하지 않는 경우에만 적용한다.

---

## CORS 설정

브라우저에서 API Gateway를 직접 호출할 때 CORS 처리가 필요하다. API Gateway 콘솔에서 CORS를 활성화하면 `OPTIONS` 메서드와 관련 헤더를 자동으로 추가해준다.

HTTP API는 콘솔에서 원클릭으로 CORS를 설정할 수 있어 편리하다. REST API는 각 리소스에 `OPTIONS` 메서드를 직접 추가하고 응답 헤더를 설정해야 한다.

```
Access-Control-Allow-Origin: https://example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
```

---

## Lambda + API Gateway 서버리스 패턴

가장 흔한 아키텍처는 API Gateway를 진입점으로 두고 각 엔드포인트가 Lambda 함수를 호출하는 구조다.

```
클라이언트
  │
  ▼
API Gateway (인증, 속도 제한, 요청 검증)
  │
  ├── GET  /users        → Lambda: listUsers
  ├── POST /users        → Lambda: createUser
  ├── GET  /users/{id}   → Lambda: getUser
  └── PUT  /users/{id}   → Lambda: updateUser
              │
              ▼
           DynamoDB / RDS
```

이 패턴에서 Lambda는 서버 없이 코드만 실행되고, API Gateway가 HTTP 계층을 담당한다. 인프라 관리 부담이 크게 줄어드는 대신, 콜드 스타트(Cold Start)와 Lambda 실행 시간 제한(최대 29초, API Gateway 타임아웃)을 고려해야 한다.

---

## CloudWatch 통합 모니터링

API Gateway는 기본적으로 CloudWatch에 지표를 전송한다.

| 지표 | 설명 |
|------|------|
| Count | 총 API 요청 수 |
| 4XXError | 클라이언트 오류 수 |
| 5XXError | 서버 오류 수 |
| Latency | 요청부터 응답까지 전체 시간 |
| IntegrationLatency | API Gateway → 백엔드 응답 시간 |
| CacheHitCount | 캐시 히트 수 (REST API, 캐싱 활성화 시) |
| CacheMissCount | 캐시 미스 수 (REST API, 캐싱 활성화 시) |

`Latency - IntegrationLatency` 차이가 크면 API Gateway 자체의 오버헤드가 크다는 의미다. 이럴 때는 HTTP API로 전환하거나 매핑 템플릿을 단순화한다.

---

## REST API vs HTTP API 선택 기준

| 기능 | REST API | HTTP API |
|------|----------|----------|
| 가격 | 상대적으로 높음 | ~70% 저렴 |
| 지연 시간 | 보통 | 낮음 |
| 캐싱 | 지원 | 미지원 |
| API Key / Usage Plan | 지원 | 미지원 |
| 요청 검증 | 지원 | 미지원 |
| Lambda Authorizer | 지원 | 지원 |
| Cognito Authorizer | 지원 | 지원 (JWT) |
| OIDC / OAuth 2.0 | 직접 구현 필요 | 기본 지원 |
| WebSocket | 별도 WebSocket API | 미지원 |
| AWS 서비스 직접 통합 | 지원 | 미지원 |

**정리**: 단순한 Lambda 프록시나 외부 HTTP 연동이라면 HTTP API, 캐싱·Usage Plan·요청 검증·AWS 서비스 직접 통합이 필요하다면 REST API를 선택한다.

---

## 정리

- API Gateway는 REST API, HTTP API, WebSocket API 세 가지 유형을 제공한다
- Lambda 프록시 통합이 가장 일반적이며, Lambda가 `statusCode`, `headers`, `body`를 포함한 객체를 반환한다
- 인증은 IAM, Cognito, Lambda Authorizer, API Key 네 가지 방식을 지원한다
- 속도 제한은 계정/리전 레벨과 스테이지/메서드 레벨 두 단계로 적용된다
- 캐싱은 REST API에서만 지원되며, 반복 요청이 많은 읽기 전용 엔드포인트에 효과적이다
- 대부분의 새 프로젝트에는 저렴하고 빠른 HTTP API가 더 적합하다
