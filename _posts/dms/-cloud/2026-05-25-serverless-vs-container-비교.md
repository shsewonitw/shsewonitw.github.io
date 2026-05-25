---
layout: post
title: "[Daily morning study] Serverless vs Container 비교"
description: >
  #daily morning study
category: 
    - dms
    - dms-cloud
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 개요

Serverless와 Container는 현대 클라우드 애플리케이션 배포의 두 가지 주요 패러다임이다. 각각의 특성을 이해하고 상황에 맞게 선택하는 것이 중요하다.

---

## Container

컨테이너는 애플리케이션과 그 실행 환경(라이브러리, 설정 파일 등)을 하나의 이미지로 패키징한 독립적인 실행 단위다.

### 핵심 특징

- **이식성**: 동일한 이미지가 어떤 환경에서든 같은 방식으로 동작
- **일관성**: 개발·테스트·프로덕션 환경을 완전히 동일하게 유지
- **오케스트레이션**: Kubernetes로 대규모 클러스터 관리 가능
- **장기 실행에 적합**: 항상 켜져 있는 서버 프로세스에 자연스럽게 맞음

### 주요 기술

- **Docker**: 컨테이너 빌드 및 실행
- **Kubernetes (K8s)**: 컨테이너 오케스트레이션
- **AWS ECS / EKS, GKE, AKS**: 관리형 쿠버네티스 서비스

### Dockerfile 예시 (Node.js)

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

```bash
docker build -t my-app .
docker run -p 3000:3000 my-app
```

---

## Serverless

Serverless는 서버 관리 없이 코드(함수)를 실행하는 패러다임이다. 클라우드 프로바이더가 인프라를 완전히 관리하고, 개발자는 비즈니스 로직에만 집중한다.

### 핵심 특징

- **이벤트 기반**: HTTP 요청, 큐 메시지, 타이머 등으로 트리거
- **자동 스케일링**: 요청량에 따라 0 → 수천 개로 자동 확장·축소
- **사용량 기반 과금**: 함수 실행 시간·횟수만큼만 비용 발생
- **콜드 스타트**: 비활성 상태에서 첫 요청 시 초기화 지연 발생

### 주요 서비스

- AWS Lambda
- Google Cloud Functions
- Azure Functions
- Cloudflare Workers

### Lambda 함수 예시 (Node.js)

```javascript
exports.handler = async (event) => {
    const { name } = JSON.parse(event.body);

    return {
        statusCode: 200,
        body: JSON.stringify({ message: `Hello, ${name}!` }),
    };
};
```

---

## 비교 정리

| 항목 | Container | Serverless |
|------|-----------|------------|
| 서버 관리 | 직접 관리 (또는 관리형) | 클라우드 프로바이더가 전담 |
| 스케일링 | 수동 설정 / HPA | 자동 (0~무한대) |
| 과금 방식 | 실행 중인 인스턴스 기준 | 함수 실행 시간·횟수 기준 |
| 콜드 스타트 | 없음 (항상 실행 중) | 있음 (비활성 후 첫 요청 시 지연) |
| 실행 시간 제한 | 없음 | 있음 (Lambda 최대 15분) |
| 상태 유지 | 가능 | 기본적으로 stateless |
| 환경 커스터마이징 | 높음 | 낮음 (런타임 제한) |
| 초기 설정 복잡도 | 높음 | 낮음 |

---

## 언제 어떤 걸 선택할까

### Container가 적합한 경우

- 웹 서버, API 서버처럼 항상 켜져 있어야 하는 장기 실행 프로세스
- 복잡한 의존성이나 커스텀 런타임이 필요한 경우
- 실행 시간이 15분을 넘는 배치 작업
- 세밀한 인프라 제어가 필요할 때
- 레거시 시스템을 점진적으로 클라우드로 이전할 때

### Serverless가 적합한 경우

- 파일 업로드 후 이미지 리사이징, 이메일 발송처럼 이벤트 기반 처리
- 트래픽이 불규칙하거나 예측하기 어려운 경우
- 빠른 프로토타이핑과 MVP 개발
- 실행 시간이 짧은 API 엔드포인트
- 크론 잡(스케줄 기반 작업)

---

## 콜드 스타트 문제와 해결 방법

Serverless의 가장 큰 단점은 콜드 스타트다. 함수가 일정 시간 비활성 상태였다가 요청이 들어오면 컨테이너 초기화에 수백 ms~수 초의 지연이 발생한다.

### 해결 방법

1. **Provisioned Concurrency (AWS Lambda)**: 인스턴스를 미리 워밍업해 대기 상태로 유지
2. **주기적인 Warm-up 요청**: CloudWatch 이벤트로 더미 요청을 보내 활성 유지
3. **가벼운 런타임 선택**: Python, Node.js는 Java·.NET보다 초기화가 훨씬 빠름
4. **의존성 최소화**: 패키지 크기를 줄여 초기화 시간 단축

```javascript
// 핸들러 밖에서 DB 클라이언트를 초기화하면 재사용 가능
// 콜드 스타트 시에만 생성되고, 이후 요청에서는 재사용됨
const dbClient = new DatabaseClient(process.env.DB_URL);

exports.handler = async (event) => {
    const result = await dbClient.query('SELECT * FROM users');
    return { statusCode: 200, body: JSON.stringify(result) };
};
```

---

## 하이브리드 아키텍처

실제 프로덕션에서는 Container와 Serverless를 함께 사용하는 경우가 많다.

- **Kubernetes** 위에서 메인 웹 서버 실행 → Container
- **Lambda**로 이미지 처리, 알림 발송 → Serverless
- **AWS Fargate**로 배치 작업 실행 → Container처럼 쓰지만 서버 관리 불필요

Fargate는 Container와 Serverless의 중간 형태다. 컨테이너 이미지 그대로 실행하면서도 서버 프로비저닝이 필요 없다. Google Cloud Run도 같은 방식으로 동작한다.

---

## 정리

Container와 Serverless는 대립 관계가 아니라 상호 보완적이다. 장기 실행·복잡한 환경이 필요하면 Container를, 이벤트 기반의 짧은 실행·예측 불가한 트래픽에는 Serverless를 선택한다. Fargate, Cloud Run 같은 서비스가 등장하면서 두 패러다임의 경계는 점점 흐려지고 있다. 중요한 건 서비스의 특성에 맞게 도구를 고르는 것이다.
