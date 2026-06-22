---
layout: post
title: "[Daily morning study] 데이터베이스 복제(Replication) 전략"
description: >
  #daily morning study
category: 
    - dms
    - dms-database
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 복제(Replication)란?

데이터베이스 복제는 하나의 DB 서버에 있는 데이터를 다른 서버로 자동으로 동기화하는 메커니즘이다. 가용성, 읽기 성능, 장애 복구를 목적으로 사용한다.

복제를 구성하면 데이터를 쓰는 서버와 읽는 서버를 분리할 수 있고, 특정 서버가 장애가 나더라도 서비스를 지속할 수 있다.

---

## 마스터-슬레이브(Master-Slave) 복제

가장 흔하게 쓰이는 구조다. 하나의 **Primary(Master)** 서버에만 쓰기가 허용되고, 변경 사항이 하나 이상의 **Replica(Slave)** 서버로 전파된다.

```
Client Write → [Primary] → 복제 로그 → [Replica 1]
                                      → [Replica 2]
Client Read  → [Replica 1 or Replica 2]
```

### 동작 원리

MySQL 기준으로 Primary는 변경 사항을 **바이너리 로그(Binary Log)**에 기록하고, Replica는 이 로그를 읽어서 동일한 SQL을 재실행하거나 row 변경 내용을 그대로 적용한다.

| 구성 요소 | 역할 |
|-----------|------|
| Binary Log | Primary의 변경 이력 기록 |
| I/O Thread | Replica가 Primary의 바이너리 로그를 가져오는 스레드 |
| Relay Log | Replica가 가져온 로그를 임시 저장하는 파일 |
| SQL Thread | Relay Log를 읽어 실제 DB에 반영하는 스레드 |

### 장점

- 읽기 쿼리를 Replica로 분산해 Primary 부하를 줄인다
- Replica를 백업 용도로 활용할 수 있다
- Primary 장애 시 Replica를 승격(failover)해 다운타임을 최소화한다

### 단점

- 쓰기는 반드시 Primary 한 곳에만 가능하다 → 쓰기 병목 발생 가능
- 비동기 복제 시 Primary 장애 직후 Replica에 최신 데이터가 없을 수 있다 (복제 지연)
- Replica 수가 많아질수록 Primary의 네트워크/디스크 부하가 늘어난다

---

## 마스터-마스터(Master-Master) 복제

두 서버 모두 읽기/쓰기를 처리하고, 서로 상대방의 변경 사항을 복제한다. Active-Active 구성이라고도 한다.

```
Client Write/Read → [Node A] ←→ [Node B] ← Client Write/Read
```

### 장점

- 어느 노드에서도 쓰기가 가능해 가용성이 높다
- 한 노드가 죽어도 다른 노드가 서비스를 이어받는다

### 단점

- **쓰기 충돌(Write Conflict)** 문제가 생긴다. 두 노드에서 동시에 같은 row를 수정하면 어떤 값을 최종으로 볼지 결정해야 한다.
- 충돌 해결 로직이 복잡해지고, 잘못 처리하면 데이터 불일치가 발생한다.
- 일반 RDBMS에서는 직접 충돌 정책을 구현해야 한다.

실제로 순수 Active-Active 마스터-마스터는 관리 난이도가 높아서, 실무에서는 지역(Region)을 분리해 각 지역에서만 쓰는 영역을 나누는 방식으로 충돌을 회피한다.

---

## 동기식 vs 비동기식 복제

### 비동기식(Asynchronous) 복제

Primary가 트랜잭션을 커밋하고 나서 나중에 Replica로 전파한다. MySQL의 기본 복제 방식이다.

- **장점**: 성능 부하가 적다. Replica 응답을 기다리지 않아 빠르다.
- **단점**: Primary 장애 시 Replica가 아직 반영 못 한 데이터가 유실될 수 있다 (RPO > 0).

### 동기식(Synchronous) 복제

Primary가 커밋할 때 최소 하나 이상의 Replica에 데이터가 쓰여진 것을 확인한 뒤 응답한다.

- **장점**: 데이터 유실이 없다. RPO = 0.
- **단점**: 쓰기 레이턴시가 늘어난다. Replica가 느리거나 장애가 나면 Primary도 멈춘다.

### 반동기식(Semi-Synchronous) 복제

MySQL에서 지원하는 절충안이다. Primary는 하나 이상의 Replica가 바이너리 로그를 **수신**했다는 ACK를 받은 뒤 커밋한다. Replica가 실제로 반영했는지는 확인하지 않는다.

```
Primary: 트랜잭션 실행 → 바이너리 로그 작성
       → Replica ACK 대기 → ACK 수신 후 클라이언트에 응답
```

데이터 유실 가능성을 줄이면서도 완전 동기식보다 빠르다.

---

## 복제 지연(Replication Lag)

비동기 복제에서 Replica가 Primary를 따라가지 못하는 상태다. 쓰기가 폭증하거나 Replica 서버 성능이 낮을 때 발생한다.

```sql
-- MySQL에서 복제 지연 확인
SHOW REPLICA STATUS\G
-- Seconds_Behind_Source 값이 복제 지연 초 수
```

복제 지연이 크면 Replica에서 읽는 클라이언트가 오래된 데이터를 보게 된다. **Read-Your-Writes** 일관성이 필요한 경우(방금 쓴 데이터를 바로 읽어야 하는 경우)에는 해당 쿼리를 Primary에서 읽도록 라우팅해야 한다.

---

## 장애 복구 (Failover)

Primary가 다운되면 Replica 중 하나를 새 Primary로 승격해야 한다.

| 방식 | 설명 |
|------|------|
| 수동 Failover | DBA가 직접 `CHANGE MASTER TO`로 Replica를 Primary로 전환 |
| 자동 Failover | MHA, Orchestrator, ProxySQL, AWS RDS Multi-AZ 등이 자동으로 감지하고 승격 |

AWS RDS Multi-AZ는 동기식 복제로 구성된 Standby를 두고, Primary 장애 시 보통 60~120초 내에 자동으로 Standby를 Primary로 전환한다.

---

## 실무 활용 패턴

### 읽기/쓰기 분리

```
애플리케이션
  ├── 쓰기 쿼리 → Primary
  └── 읽기 쿼리 → Replica (Load Balancer로 분산)
```

Spring 환경에서는 `@Transactional(readOnly = true)` 트랜잭션을 Replica로 라우팅하도록 `AbstractRoutingDataSource`를 활용하는 패턴이 자주 쓰인다.

### 분석 쿼리 분리

무거운 분석 쿼리(OLAP 성격)를 별도 Replica에서 돌리면 서비스용 Primary와 Replica에 영향을 주지 않는다.

### 지역 분산(Geo-Replication)

서울 리전의 Primary → 도쿄 리전의 Replica로 복제해두면, 재해 시 도쿄 리전으로 트래픽을 전환할 수 있다.

---

## 정리

| 구분 | 마스터-슬레이브 | 마스터-마스터 |
|------|----------------|--------------|
| 쓰기 가능 서버 | 1개 (Primary) | 2개 이상 |
| 충돌 문제 | 없음 | 있음 (해결 필요) |
| 주 용도 | 읽기 확장, 장애 복구 | 고가용성, 지역 분산 |
| 관리 복잡도 | 낮음 | 높음 |

복제는 가용성과 성능을 높이는 핵심 기술이지만, 데이터 일관성 트레이드오프를 항상 고려해야 한다. 어떤 복제 방식을 선택하느냐는 서비스의 RPO(복구 목표 시간), 쓰기 부하, 운영 역량에 따라 결정된다.
