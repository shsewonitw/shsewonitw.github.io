---
layout: post
title: "[Daily morning study] 클라우드 네이티브 애플리케이션과 12-Factor App 원칙"
description: >
  #daily morning study
category: 
    - dms
    - dms-cloud
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 클라우드 네이티브란?

클라우드 네이티브(Cloud Native)는 단순히 클라우드에서 실행되는 앱이 아니라, 클라우드 환경의 특성을 최대한 활용하도록 처음부터 설계된 애플리케이션을 가리킨다. CNCF(Cloud Native Computing Foundation)는 클라우드 네이티브를 이렇게 정의한다.

> 컨테이너, 마이크로서비스, 동적 오케스트레이션, 지속적 배포를 활용해 느슨하게 결합되고, 탄력적이며, 관리 가능하고, 관찰 가능한 시스템을 구축하는 방식

핵심 특성:

- **컨테이너 기반**: Docker + Kubernetes로 환경 일관성 확보
- **마이크로서비스 구조**: 서비스별 독립 배포와 확장 가능
- **API 우선 설계**: 서비스 간 통신은 명확한 API 계약으로
- **자동화된 CI/CD**: 빠르고 안전한 배포 파이프라인
- **옵저버빌리티**: Metrics, Tracing, Logging으로 상태 파악

---

## 12-Factor App이란?

12-Factor App은 Heroku 개발자들이 정리한 SaaS 애플리케이션 구축 방법론으로, 클라우드 네이티브 앱의 사실상 표준 지침이 됐다. 12개의 원칙으로 구성되며, 각각이 현대 앱 개발에서 자주 발생하는 문제를 해결한다.

---

## 12가지 원칙 상세

### I. Codebase — 하나의 코드베이스, 여러 배포

```
[코드베이스]
    │
    ├── dev 환경
    ├── staging 환경
    └── production 환경
```

- 1개의 Git 저장소 → 여러 환경(dev/staging/prod)에 배포
- 공유 코드는 라이브러리로 분리, 별도 저장소로 관리
- 앱 1개 = 저장소 1개 원칙 유지

**안 좋은 패턴**: prod 브랜치, dev 브랜치를 완전히 다른 코드로 관리

---

### II. Dependencies — 의존성 명시와 격리

- 암묵적인 시스템 의존성 금지. 모든 의존성은 명시적으로 선언
- 언어별 패키지 매니저 활용

```bash
# Node.js
package.json + package-lock.json

# Python
requirements.txt 또는 pyproject.toml

# Go
go.mod + go.sum
```

- 의존성 격리: 가상환경(venv), Docker 컨테이너로 시스템 패키지 오염 방지

---

### III. Config — 설정은 환경 변수로

코드와 설정(config)을 분리하는 것이 핵심이다.

| 나쁜 예 | 좋은 예 |
|---------|---------|
| DB URL을 코드에 하드코딩 | `DATABASE_URL` 환경 변수로 주입 |
| API 키를 `config.json`에 커밋 | `.env` 파일 + `.gitignore` |

```bash
# .env 파일 예시 (로컬 전용, 절대 커밋하지 않음)
DATABASE_URL=postgres://user:pass@localhost:5432/mydb
SECRET_KEY=supersecret
DEBUG=true
```

환경마다 다른 설정(DB 주소, 로그 레벨, 외부 API 키)은 환경 변수로만 관리해야 한다. Kubernetes에서는 `ConfigMap`과 `Secret`이 이 역할을 맡는다.

---

### IV. Backing Services — 연결된 서비스는 리소스로 취급

백엔드 서비스(DB, 캐시, 메시지 큐, 외부 API)는 URL/자격증명만 바꾸면 교체 가능한 외부 리소스로 취급한다.

```
앱 ──── DB (로컬 MySQL)
         ↓ URL만 변경
앱 ──── DB (AWS RDS)
```

로컬 PostgreSQL이든 Amazon RDS든 앱 코드 변경 없이 환경 변수만 바꿔서 전환 가능해야 한다.

---

### V. Build, Release, Run — 빌드/릴리즈/실행 단계 엄격히 분리

```
[코드] → Build → [빌드 산출물] → Release → [릴리즈] → Run → [실행]
                                    + 설정 주입
```

- **Build**: 소스 코드를 실행 가능한 번들로 컴파일 (의존성 포함)
- **Release**: 빌드 산출물 + 환경 설정 결합. 불변(immutable)이어야 함
- **Run**: 릴리즈를 특정 환경에서 실행

릴리즈는 번호나 타임스탬프로 식별되고, 한 번 생성된 릴리즈는 수정 불가. 롤백도 이전 릴리즈로 되돌리는 방식이다.

---

### VI. Processes — 앱은 하나 이상의 무상태 프로세스로 실행

- 프로세스는 무상태(stateless), 비공유(share-nothing)여야 한다
- 요청 간에 상태를 메모리에 저장하면 안 됨
- 세션, 캐시 같은 상태는 외부 저장소(Redis, DB)에 유지

```
❌ 안 좋음: 세션 정보를 앱 서버 메모리에 저장
           → 스케일 아웃 시 다른 인스턴스에서 세션 없음

✅ 좋음: 세션을 Redis에 저장
        → 어느 인스턴스가 요청 처리해도 동일한 세션 접근 가능
```

---

### VII. Port Binding — 포트 바인딩으로 서비스 공개

앱은 외부 웹 서버(Apache, Nginx)에 의존하지 않고, 자체적으로 HTTP 서비스를 포트에 바인딩해서 공개한다.

```javascript
// Express.js 예시
const app = express();
app.listen(process.env.PORT || 3000);
```

```python
# FastAPI 예시
uvicorn.run(app, host="0.0.0.0", port=int(os.environ.get("PORT", 8000)))
```

포트 번호도 환경 변수로 주입받는다.

---

### VIII. Concurrency — 프로세스 모델로 수평 확장

확장이 필요할 때 프로세스를 수평으로 늘린다.

```
웹 프로세스 × 3   (HTTP 요청 처리)
워커 프로세스 × 5 (백그라운드 작업)
```

Kubernetes의 Deployment `replicas` 설정이 이 원칙을 그대로 구현한다. 프로세스 내부 스레드 수보다 프로세스 자체를 늘리는 방식으로 스케일 아웃한다.

---

### IX. Disposability — 빠른 시작과 그레이스풀 셔다운

- **빠른 시작**: 프로세스는 가능한 빨리 시작돼야 함 (오토스케일링 반응 속도)
- **그레이스풀 셔다운**: SIGTERM 수신 시 진행 중인 요청을 완료하고 정상 종료

```javascript
process.on('SIGTERM', () => {
  server.close(() => {
    // DB 연결 정리 등
    process.exit(0);
  });
});
```

Kubernetes에서 Pod 종료 시 이 원칙이 지켜지지 않으면 요청 유실이 발생한다.

---

### X. Dev/Prod Parity — 개발과 운영 환경의 차이를 최소화

| 차이 유형 | 문제 |
|-----------|------|
| 시간 차이 | 코드 작성 후 배포까지 수 주 소요 |
| 인원 차이 | 개발자 ≠ 운영자 |
| 도구 차이 | 개발은 SQLite, 운영은 PostgreSQL |

"개발에서 잘 됐는데 운영에서 왜 안 되지?" 문제의 근본 원인이다. Docker Compose로 로컬에서도 운영과 동일한 스택(PostgreSQL, Redis 등)을 사용하는 것이 해결책이다.

---

### XI. Logs — 로그는 이벤트 스트림으로 취급

앱은 로그 파일 관리에 관여하지 않는다. 로그를 stdout으로 출력하고, 로그 수집은 인프라가 담당한다.

```python
import sys
print("서버 시작", file=sys.stdout)
```

```
앱 (stdout) → Fluentd/Filebeat → Elasticsearch → Kibana
앱 (stdout) → CloudWatch Logs → 분석/알림
```

로그 파일 경로 설정, 로그 로테이션 설정을 앱 코드에서 하지 않는다.

---

### XII. Admin Processes — 관리 작업은 일회성 프로세스로 실행

DB 마이그레이션, 데이터 픽스, 콘솔 접속 같은 관리 작업은 정규 앱 프로세스와 동일한 환경에서 일회성으로 실행한다.

```bash
# 나쁜 패턴: 운영 DB에 직접 접속해서 SQL 실행
# 좋은 패턴: 마이그레이션 스크립트를 배포 파이프라인에 포함

# Kubernetes에서 마이그레이션 Job 실행
kubectl apply -f db-migration-job.yaml
```

---

## 12-Factor App 요약표

| # | 원칙 | 핵심 |
|---|------|------|
| I | Codebase | 1저장소 → N배포 |
| II | Dependencies | 의존성 명시·격리 |
| III | Config | 환경 변수로 설정 관리 |
| IV | Backing Services | 외부 서비스는 교체 가능한 리소스 |
| V | Build/Release/Run | 단계 엄격히 분리 |
| VI | Processes | 무상태 프로세스 |
| VII | Port Binding | 앱 자체가 포트 바인딩 |
| VIII | Concurrency | 프로세스 수평 확장 |
| IX | Disposability | 빠른 시작·그레이스풀 종료 |
| X | Dev/Prod Parity | 환경 차이 최소화 |
| XI | Logs | stdout 이벤트 스트림 |
| XII | Admin Processes | 관리 작업은 일회성 프로세스 |

---

## 클라우드 네이티브 기술 스택과의 연결

12-Factor App 원칙이 현대 클라우드 기술과 어떻게 연결되는지 보면:

```
III  Config           → Kubernetes ConfigMap / Secret
VI   Processes        → Kubernetes Deployment (stateless pods)
VIII Concurrency      → Kubernetes HPA (수평 자동 확장)
IX   Disposability    → Kubernetes graceful shutdown, probe
X    Dev/Prod Parity  → Docker Compose (로컬), Helm (운영)
XI   Logs             → Fluentd + ELK Stack / CloudWatch
XII  Admin Processes  → Kubernetes Job / CronJob
```

12-Factor App은 컨테이너 이전에 만들어진 방법론이지만, Kubernetes 생태계와 거의 완벽하게 맞아떨어진다.

---

## 실무에서의 적용 포인트

**환경 변수 주입 패턴 (Kubernetes)**:

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: app
          image: myapp:1.0
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: url
            - name: LOG_LEVEL
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: log_level
```

**그레이스풀 셔다운 (Node.js)**:

```javascript
const server = app.listen(PORT);

const shutdown = () => {
  server.close(() => {
    dbPool.end(); // DB 커넥션 반환
    process.exit(0);
  });
  setTimeout(() => process.exit(1), 10000); // 10초 후 강제 종료
};

process.on('SIGTERM', shutdown);
process.on('SIGINT', shutdown);
```

---

## 정리

12-Factor App은 2011년에 나온 방법론이지만 지금도 유효하다. 마이크로서비스, 쿠버네티스, 서버리스 등 어떤 클라우드 환경에서도 이 원칙을 따르면 다음을 얻는다:

- 환경 이동성: 로컬 → dev → staging → prod 간 코드 변경 없이 이동
- 수평 확장성: 무상태 프로세스로 언제든 인스턴스 추가/제거 가능
- 운영 안정성: 빠른 시작, 그레이스풀 종료, 명확한 로그
- 팀 생산성: 개발/운영 환경 통일로 "내 로컬에선 됐는데" 문제 감소

현재 프로젝트가 12가지 원칙 중 몇 개를 지키는지 점검해보는 것만으로도 개선 포인트를 찾을 수 있다.
