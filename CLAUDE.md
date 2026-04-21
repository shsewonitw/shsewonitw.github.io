# CLAUDE.md — Blog Post Creation Guidelines

## Post File Naming Rules

When creating a new post file under `_posts/`, the **filename** must follow these rules to avoid `invalid path` errors on Windows during `git pull`.

### Forbidden characters in filenames

| Character | Reason | Replacement |
|-----------|--------|-------------|
| `:` (colon) | Invalid on Windows | remove or replace with `-` |
| `?` (question mark) | Invalid on Windows | remove |
| `/` (slash) | Creates unintended nested directories | replace with `-` |
| `\` (backslash) | Invalid on Windows | replace with `-` |
| `*` `<` `>` `"` `\|` | Invalid on Windows | remove |

### Example

**Title in front matter** (special characters allowed):
```
title: "Transformer 아키텍처의 핵심: 셀프 어텐션(Self-Attention)의 원리"
```

**Filename** (special characters must be removed/replaced):
```
# Bad — causes git pull error
2025-11-13-transformer-아키텍처의-핵심:-셀프-어텐션(self-attention)의-원리.md

# Good
2025-11-13-transformer-아키텍처의-핵심-셀프-어텐션(self-attention)의-원리.md
```

More examples:

| Original title fragment | Filename fragment |
|------------------------|------------------|
| `무엇인가?` | `무엇인가` |
| `하는가?)` | `하는가)` |
| `big-o 표기법: 시간 복잡도` | `big-o-표기법-시간-복잡도` |
| `CI/CD 파이프라인` | `cicd-파이프라인` or `ci-cd-파이프라인` |
| `TCP/IP 4계층` | `tcpip-4계층` or `tcp-ip-4계층` |
| `비동기 I/O` | `비동기-io` |
| `HTTP/1.1, HTTP/2` | `http-1.1-http-2` |

### Checklist before saving a new post file

- [ ] Filename contains no `:` `?` `/` `\` `*` `<` `>` `"` `|`
- [ ] Filename format: `YYYY-MM-DD-slug.md`
- [ ] `title:` in front matter may still contain the original special characters for display

## Category Slugs

The `category:` field in post front matter must use the exact slugs below.
Do NOT use the old `-ai`, `-backend`, etc. values — they cause 404 errors.

### DMS sub-categories

| Title shown in sidebar | Slug to use in `category:` |
|------------------------|---------------------------|
| AI                     | `dms-ai`                  |
| Frontend               | `dms-frontend`            |
| Backend                | `dms-backend`             |
| Devops                 | `dms-devops`              |
| Cloud                  | `dms-cloud`               |
| DSA                    | `dms-dsa`                 |
| OS                     | `dms-os`                  |
| Network                | `dms-network`             |
| Database               | `dms-database`            |

**Rule:** DMS sub-category slugs always follow the pattern `dms-<topic>`.

## Post Front Matter Template

```yaml
---
layout: post
title: "[Daily morning study] 제목 (특수문자 허용)"
description: >
  #daily morning study
category: 
    - dms
    - dms-ai        # ← use dms-<topic> slug from the table above
hide_last_modified: true
---
```

## Duplicate Post Prevention

**Before writing a new post, always check existing posts to avoid duplicate topics.**

```bash
# Check existing post titles
ls _posts/dms/-database/
ls _posts/dms/-ai/
# (check the relevant category directory)
```

### Rules

- Do NOT write a post whose core topic is already covered by an existing post.
- If a related post exists, choose a clearly different angle or a more specific subtopic.
- Redis posts already exist covering: data structures, caching strategies (Cache-Aside, Write-Through, Write-Behind), cache invalidation, TTL, eviction policy, persistence (RDB/AOF).

### Bad examples (duplicate)

| Existing post | Bad new post (too similar) |
|--------------|---------------------------|
| Redis 캐싱 전략과 핵심 자료구조 | Redis 데이터 구조와 캐싱 전략 ← same content |
| Redis 캐시 전략 (Cache-Aside...) | Redis 캐시 전략 정리 ← same content |

### Good examples (distinct angle)

| Existing post | Good new post (different angle) |
|--------------|--------------------------------|
| Redis 캐싱 전략 | Redis Pub/Sub과 Stream 활용 |
| Redis 데이터 구조 | Redis Cluster 구성과 Failover |

## Already Covered Topics (as of 2026-04)

Check these before picking a new topic. Do NOT repeat these.

### dms-ai
- RAG (Retrieval-Augmented Generation) 개념과 작동 방식
- LLM 파인튜닝과 프롬프트 엔지니어링 차이점
- Transformer 아키텍처와 셀프 어텐션 원리
- Vector Database 개념과 Embedding 관계

### dms-frontend
- TypeScript Utility Types (Partial, Pick, Omit)
- React Server Components (RSC) 개념과 장점
- Next.js App Router vs Pages Router
- Svelte / SvelteKit가 React와 다른 점
- WebAssembly (WASM) 개념과 사용 사례

### dms-backend
- Node.js 이벤트 루프와 비동기 I/O
- gRPC와 REST API 차이점
- Microservice Architecture (MSA) 장단점
- Go 고루틴과 채널
- Serverless 아키텍처와 AWS Lambda

### dms-devops
- Docker와 Kubernetes 역할 차이
- Ansible과 Terraform 차이 (CM vs Provisioning)
- Prometheus와 Grafana 모니터링
- Terraform IaC
- Jenkins vs GitHub Actions
- Observability 3요소 (Metrics, Tracing, Logging)
- GitHub Actions CI/CD 파이프라인

### dms-cloud
- IaaS, PaaS, SaaS 차이점
- VPC 개념과 서브넷, 라우팅
- AWS IAM 중요성
- 로드 밸런서 종류와 작동 방식 (L4 vs L7)
- CDN 작동 원리와 장점
- Auto Scaling 그룹 원리
- AWS EC2, S3, RDS 기본 개념
- AWS, GCP, Azure 3사 비교

### dms-dsa
- B-Tree와 B+Tree 차이점
- Big-O 표기법, 시간/공간 복잡도
- 동적 계획법 (Dynamic Programming)
- 해시 테이블 작동 원리와 충돌 해결
- BFS와 DFS 차이점 및 활용

### dms-os
- 가상 메모리와 페이징
- Deadlock 4가지 발생 조건과 해결 방법
- CPU 스케줄링 알고리즘 (FIFO, SJF, Round Robin)
- 프로세스와 스레드 차이점
- Mutex와 Semaphore 차이

### dms-network
- TCP 3-way / 4-way Handshake
- OSI 7계층과 TCP/IP 4계층 모델
- DNS 작동 원리 (Recursive vs Iterative)
- HTTP/1.1, HTTP/2, HTTP/3 차이점
- RESTful API 6가지 원칙

### dms-database
- 데이터베이스 인덱스 작동 원리와 장단점
- 트랜잭션 ACID 속성
- 트랜잭션 격리 수준 (Isolation Levels)
- SQL vs NoSQL 비교
- 데이터베이스 정규화 (1NF, 2NF, 3NF)
- Redis 캐시 전략 (Cache-Aside, Write-Through, Write-Behind, 캐시 무효화)
- Redis 데이터 구조와 캐싱 전략 (자료구조, TTL, Eviction, Persistence, Cluster)

## Topic Diversity Guidelines

To keep the blog well-rounded, follow these rules when selecting a new topic:

### Rotate across categories
Do not write posts in the same category on consecutive days.
Aim for a rotation like: AI → Backend → Network → OS → DSA → Frontend → Devops → Cloud → Database → repeat.

### Suggested next topics (not yet covered)

**dms-ai**
- GPT 모델 동작 방식과 토크나이저 이해
- Diffusion 모델 원리 (Stable Diffusion)
- AI Agent와 Tool Use 개념
- RLHF (Reinforcement Learning from Human Feedback)
- MCP (Model Context Protocol) 개념과 활용

**dms-frontend**
- 브라우저 렌더링 과정 (Critical Rendering Path)
- 가상 DOM (Virtual DOM) 작동 원리
- CSS-in-JS vs CSS Modules 비교
- 웹 성능 최적화 (Lazy Loading, Code Splitting)
- PWA (Progressive Web App) 개념과 구현

**dms-backend**
- JWT vs Session 인증 방식 비교
- 메시지 큐 (Kafka, RabbitMQ) 개념과 활용
- 데이터베이스 커넥션 풀링
- API Rate Limiting 구현 방법
- WebSocket vs Server-Sent Events

**dms-devops**
- Blue-Green 배포와 Canary 배포 전략
- Kubernetes HPA (Horizontal Pod Autoscaler)
- Docker 이미지 최적화 (멀티스테이지 빌드)
- GitOps와 ArgoCD 개념
- 시크릿 관리 (HashiCorp Vault, AWS Secrets Manager)

**dms-cloud**
- AWS SQS, SNS 차이점과 활용
- 클라우드 비용 최적화 전략
- Serverless vs Container 비교
- AWS Route 53과 DNS 라우팅 정책
- 멀티 리전 아키텍처 설계

**dms-dsa**
- 트리 순회 알고리즘 (Pre/In/Post-order)
- 최단 경로 알고리즘 (Dijkstra, Bellman-Ford)
- 정렬 알고리즘 비교 (Quick, Merge, Heap Sort)
- 그리디 알고리즘 개념과 예시
- 슬라이딩 윈도우 / 투 포인터 기법

**dms-os**
- 메모리 단편화와 가비지 컬렉션
- 파일 시스템 구조 (inode, 디렉토리 트리)
- 인터럽트와 시스템 콜 동작 방식
- IPC (Inter-Process Communication) 방법들
- 캐시 메모리 계층 구조 (L1/L2/L3)

**dms-network**
- HTTPS와 TLS 핸드셰이크 과정
- 웹소켓 프로토콜 동작 방식
- NAT와 포트 포워딩
- CORS 개념과 해결 방법
- 로드 밸런싱 알고리즘 (Round Robin, Least Connections)

**dms-database**
- Redis Pub/Sub과 Stream 활용
- Redis Cluster 구성과 Failover
- PostgreSQL vs MySQL 비교
- 샤딩(Sharding)과 파티셔닝 전략
- ORM의 N+1 문제와 해결 방법
