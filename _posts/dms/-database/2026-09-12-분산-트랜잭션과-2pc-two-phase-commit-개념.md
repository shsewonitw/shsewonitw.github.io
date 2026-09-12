---
layout: post
title: "[Daily morning study] 분산 트랜잭션과 2PC (Two-Phase Commit) 개념"
description: >
  #daily morning study
category: 
    - dms
    - dms-database
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 분산 트랜잭션이란?

단일 DB에서의 트랜잭션은 ACID를 로컬에서 보장할 수 있다. 하지만 MSA 환경처럼 여러 데이터베이스 노드에 걸쳐 하나의 비즈니스 연산을 수행해야 하는 경우엔 얘기가 달라진다.

예를 들어 A 계좌에서 출금하고, B 계좌에 입금하는 이체 연산에서 두 계좌가 서로 다른 DB 서버에 있다면, 출금 커밋 이후 입금 커밋 직전에 서버가 죽는 상황이 생길 수 있다. 이를 **원자성(Atomicity) 문제**라고 한다.

분산 트랜잭션은 **여러 독립 노드에 걸친 연산을 하나의 원자적 트랜잭션으로 처리**하는 메커니즘이다.

---

## 2PC (Two-Phase Commit)

2PC는 분산 트랜잭션에서 원자성을 보장하는 가장 고전적이고 널리 알려진 프로토콜이다. 이름 그대로 두 단계(Phase)로 진행된다.

### 참여자 구성

| 역할 | 설명 |
| --- | --- |
| **Coordinator (조정자)** | 트랜잭션을 시작한 노드. 전체 과정을 주도한다. |
| **Participant (참여자)** | 실제 데이터를 가지고 있는 노드들. Coordinator의 지시를 따른다. |

---

### Phase 1: Prepare (준비 단계)

1. Coordinator가 모든 Participant에게 `PREPARE` 메시지를 보낸다.
2. 각 Participant는 트랜잭션을 실제로 커밋할 수 있는지 검사한다.
   - 로컬 로그(WAL)에 변경 내용을 기록
   - 락(Lock) 획득
3. 가능하면 `YES`, 불가능하면 `NO`를 Coordinator에게 응답한다.

```
Coordinator  -->  Participant A: PREPARE
Coordinator  -->  Participant B: PREPARE

Participant A  -->  Coordinator: YES (준비 완료)
Participant B  -->  Coordinator: YES (준비 완료)
```

---

### Phase 2: Commit 또는 Abort

- **모든 Participant가 YES 응답** → Coordinator가 `COMMIT` 메시지를 전송, 각 Participant가 커밋 후 락 해제
- **하나라도 NO 응답 또는 타임아웃** → Coordinator가 `ABORT` 메시지를 전송, 모든 Participant가 롤백

```
(모두 YES인 경우)
Coordinator  -->  Participant A: COMMIT
Coordinator  -->  Participant B: COMMIT

(B가 NO인 경우)
Coordinator  -->  Participant A: ABORT
Coordinator  -->  Participant B: ABORT
```

---

## 2PC의 문제점

### 1. 블로킹 프로토콜

Phase 1에서 YES를 응답한 Participant는 Coordinator로부터 Phase 2 메시지를 받을 때까지 **락을 유지**해야 한다. 이 대기 시간 동안 해당 리소스를 다른 트랜잭션이 사용할 수 없다.

### 2. Coordinator 단일 장애점 (SPOF)

Coordinator가 Phase 1 이후 Phase 2 직전에 죽으면, Participant들은 `YES`를 응답했지만 커밋 여부를 모른 채 **락을 붙든 상태로 무한 대기**할 수 있다. 이를 **불확실 구간(In-doubt period)** 이라고 한다.

### 3. 네트워크 분단 (Network Partition)

CAP 정리 관점에서 2PC는 **CP** 계열이다. 일관성(C)과 내장애성(P) 중에서 **일관성을 우선**하다 보니 가용성(A)을 포기하는 상황이 생긴다.

---

## 2PC를 보완하는 방법들

### 3PC (Three-Phase Commit)

2PC의 불확실 구간 문제를 해결하기 위해 `Pre-Commit` 단계를 추가한 프로토콜. Coordinator 장애 시 Participant끼리 합의를 진행할 수 있다.

다만 구현 복잡도와 네트워크 비용이 증가하고, 네트워크 분단 상황에서는 여전히 완벽하지 않아 실무에서 잘 쓰이지는 않는다.

### Saga 패턴

MSA 환경에서 2PC 대신 자주 사용되는 대안. 각 서비스가 로컬 트랜잭션을 독립적으로 수행하고, **실패 시 이전 단계를 보상(Compensating Transaction)하는 방식**으로 결과적 일관성(Eventual Consistency)을 달성한다.

| 방식 | 설명 |
| --- | --- |
| **Choreography** | 각 서비스가 이벤트를 발행하고, 다음 서비스가 이를 구독해서 실행 |
| **Orchestration** | 중앙 Saga 오케스트레이터가 각 서비스에 명령을 내리는 방식 |

```
// Orchestration 예시
SagaOrchestrator:
  1. PaymentService.charge()  → 성공
  2. InventoryService.reserve() → 실패
  3. PaymentService.refund()  ← 보상 트랜잭션 실행
```

---

## 실무에서의 2PC

### 언제 2PC를 쓰는가?

- 금융 시스템처럼 **강한 일관성**이 절대적으로 필요한 경우
- 노드 간 네트워크가 안정적이고, 트랜잭션 처리량이 많지 않은 경우

### 언제 Saga나 다른 방식을 택하는가?

- MSA 환경에서 서비스가 많고 독립 배포가 중요한 경우
- 높은 가용성과 확장성이 필요한 경우
- 결과적 일관성(Eventual Consistency)을 수용할 수 있는 경우

---

## 관련 개념 연결

- **WAL (Write-Ahead Log)**: 2PC의 Phase 1에서 Participant가 `YES` 응답 전 WAL에 미리 기록한다. 덕분에 Coordinator로부터 COMMIT이 오면 실제 커밋을 수행할 수 있다.
- **CAP 정리**: 2PC는 일관성을 우선하는 CP 전략이다. 가용성을 높이려면 Saga 같은 AP 계열 패턴을 택해야 한다.
- **분산 락(Distributed Lock)**: 2PC의 준비 단계에서 Participant가 리소스 락을 잡는 방식은 분산 락과 유사한 개념이다.

---

## 정리

| 항목 | 내용 |
| --- | --- |
| **2PC 목적** | 분산 환경에서 원자성 보장 |
| **Phase 1** | Coordinator가 각 Participant에게 PREPARE 요청 |
| **Phase 2** | 모두 YES면 COMMIT, 하나라도 NO면 ABORT |
| **단점** | 블로킹, Coordinator SPOF, 불확실 구간 |
| **대안** | 3PC, Saga 패턴, TCC (Try-Confirm-Cancel) |

2PC는 분산 시스템에서 강한 일관성을 보장하는 핵심 프로토콜이지만, 성능과 가용성 트레이드오프가 크다. MSA 설계 시 어떤 일관성 수준이 필요한지를 먼저 결정하고 프로토콜을 선택하는 게 중요하다.
