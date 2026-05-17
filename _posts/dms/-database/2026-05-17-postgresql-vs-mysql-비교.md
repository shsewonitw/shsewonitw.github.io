---
layout: post
title: "[Daily morning study] PostgreSQL vs MySQL 비교"
description: >
  #daily morning study
category: 
    - dms
    - dms-database
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---
# PostgreSQL vs MySQL 비교

## 개요

PostgreSQL과 MySQL은 둘 다 오픈소스 관계형 데이터베이스이지만, 설계 철학과 지원 기능에서 차이가 꽤 크다.

- **MySQL**: 속도와 단순함을 우선시해서 읽기 중심 워크로드에 강하다.
- **PostgreSQL**: 표준 준수와 확장성을 우선시해서 복잡한 쿼리와 다양한 데이터 타입에 강하다.

---

## 핵심 차이점 요약

| 항목 | PostgreSQL | MySQL |
|------|-----------|-------|
| 라이선스 | PostgreSQL License (자유도 높음) | GPL v2 (Oracle 소유) |
| SQL 표준 준수 | 매우 높음 | 중간 (일부 비표준 문법 사용) |
| 기본 스토리지 엔진 | 단일 (Heap) | 다중 (InnoDB, MyISAM 등) |
| MVCC 구현 | 자체 MVCC | InnoDB MVCC |
| JSON 지원 | JSONB (바이너리 JSON, 인덱스 지원) | JSON (텍스트 기반) |
| 배열/범위 타입 | 지원 | 미지원 |
| 풀텍스트 검색 | 지원 | 지원 (단, PostgreSQL이 더 강력) |
| 파티셔닝 | 선언적 파티셔닝 (v10+) | 지원 |
| 복제 | 스트리밍 복제, 논리적 복제 | 바이너리 로그 기반 복제 |
| 윈도우 함수 | 완전 지원 | 지원 (v8+) |
| CTE (WITH 절) | 완전 지원 | 지원 (v8+) |
| 커뮤니티/클라우드 지원 | RDS, Cloud SQL, Azure 등 | RDS, Cloud SQL, Azure 등 |

---

## 아키텍처 차이

### 프로세스 모델

- **PostgreSQL**: 연결마다 별도의 OS 프로세스를 생성한다. 메모리 격리가 확실하지만, 연결이 많아지면 오버헤드가 커진다. 이 때문에 PgBouncer 같은 커넥션 풀러를 함께 쓰는 게 일반적이다.
- **MySQL**: 스레드 기반 모델이다. 연결당 스레드를 사용하므로 프로세스 생성 비용이 없어 대량 연결에 상대적으로 유리하다.

### 스토리지 엔진

MySQL은 스토리지 엔진을 교체할 수 있는 플러그인 아키텍처를 갖고 있다. 기본값은 InnoDB인데, 예전에는 트랜잭션이 안 되는 MyISAM이 기본이었다. PostgreSQL은 단일 스토리지 엔진을 사용하는 대신 내부 구조를 더 깊이 최적화했다.

---

## MVCC 구현 방식

MVCC(Multi-Version Concurrency Control)는 읽기와 쓰기가 서로 블로킹하지 않도록 하는 핵심 기법이다.

**PostgreSQL의 MVCC**

- 레코드에 `xmin`, `xmax` 같은 트랜잭션 ID를 직접 저장한다.
- 구버전 레코드를 같은 테이블에 남겨두다가 `VACUUM` 프로세스가 주기적으로 정리한다.
- 쓰기가 많으면 dead tuple이 쌓여 VACUUM 튜닝이 중요해진다.

**MySQL InnoDB의 MVCC**

- 언두 로그(undo log)를 별도 공간에 유지한다.
- 구버전 데이터를 테이블이 아닌 undo tablespace에 보관하므로 테이블 bloat 문제가 없다.
- 대신 언두 로그가 길어지면 롤백 비용이 커진다.

---

## 데이터 타입 비교

PostgreSQL이 지원하는 타입이 훨씬 다양하다.

**PostgreSQL 고유 타입**

```sql
-- 배열 타입
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    tags TEXT[]
);
INSERT INTO orders (tags) VALUES (ARRAY['urgent', 'bulk']);

-- 범위 타입
CREATE TABLE reservations (
    id SERIAL PRIMARY KEY,
    during TSRANGE
);
INSERT INTO reservations (during) VALUES ('[2026-05-17, 2026-05-20)');

-- JSONB (바이너리 JSON, GIN 인덱스 지원)
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    attributes JSONB
);
CREATE INDEX idx_attributes ON products USING GIN (attributes);
SELECT * FROM products WHERE attributes @> '{"color": "red"}';
```

**MySQL의 JSON**

```sql
-- JSON 타입 (텍스트 기반)
CREATE TABLE products (
    id INT AUTO_INCREMENT PRIMARY KEY,
    attributes JSON
);
-- 함수형 인덱스로 특정 경로에 인덱스 생성 가능 (v5.7.8+)
ALTER TABLE products ADD INDEX idx_color ((CAST(attributes->>'$.color' AS CHAR(50))));
```

PostgreSQL의 JSONB는 저장 시 파싱해서 바이너리로 보관하기 때문에 읽기 속도가 빠르고 GIN 인덱스를 통해 JSON 내부 값으로 효율적인 검색이 된다.

---

## 트랜잭션과 격리 수준

둘 다 ACID를 보장하지만 지원하는 격리 수준에 차이가 있다.

| 격리 수준 | PostgreSQL | MySQL InnoDB |
|-----------|-----------|-------------|
| READ UNCOMMITTED | 지원 (실제로는 RC처럼 동작) | 지원 |
| READ COMMITTED | 지원 | 지원 |
| REPEATABLE READ | 지원 | 지원 (기본값) |
| SERIALIZABLE | 지원 | 지원 |

MySQL InnoDB의 기본 격리 수준은 `REPEATABLE READ`이고, PostgreSQL의 기본값은 `READ COMMITTED`다.

PostgreSQL의 `SERIALIZABLE`은 SSI(Serializable Snapshot Isolation)를 구현해서 진짜 직렬화 가능성을 보장한다. MySQL의 SERIALIZABLE은 SELECT에 공유 락을 걸어 구현하므로 성능 비용이 더 크다.

---

## 복제(Replication)

**PostgreSQL**

- **스트리밍 복제**: WAL(Write-Ahead Log)을 실시간으로 스탠바이에 전송한다. 물리적 복제라서 바이트 단위로 동일한 복사본을 만든다.
- **논리적 복제**: 테이블 단위로 변경사항을 발행/구독하는 방식이다. 다른 버전 간 복제나 선택적 복제가 가능하다.
- Patroni, repmgr 같은 외부 도구로 자동 페일오버를 구성하는 게 일반적이다.

**MySQL**

- **바이너리 로그(binlog) 기반 복제**: Statement 기반, Row 기반, Mixed 방식이 있다. Row 기반이 가장 안정적이다.
- **GTID(Global Transaction ID)**: 트랜잭션마다 고유 ID를 부여해서 복제 위치를 추적한다. 페일오버 시 복제 재구성이 쉽다.
- MySQL Router, Orchestrator 등으로 HA를 구성한다.

---

## 성능 특성

### 읽기 성능

단순한 OLTP 읽기(PK 조회, 단순 JOIN)에서는 MySQL이 조금 더 빠른 경향이 있다. 스레드 모델과 쿼리 캐시(deprecated됐지만) 덕분에 단순 쿼리 처리량이 높다.

복잡한 분석 쿼리(윈도우 함수, 복잡한 집계, 서브쿼리)에서는 PostgreSQL의 플래너가 더 정교한 실행 계획을 세운다.

### 쓰기 성능

PostgreSQL은 dead tuple 때문에 쓰기가 많은 테이블에서 VACUUM 부하가 생긴다. `autovacuum` 설정을 잘 튜닝해야 한다.

MySQL InnoDB는 언두 로그 방식이라 테이블 bloat은 없지만, 언두 로그가 길어지는 긴 트랜잭션에는 취약하다.

---

## 언제 무엇을 선택할까

**PostgreSQL을 선택하는 경우**

- 복잡한 쿼리, 윈도우 함수, CTE를 많이 쓸 때
- JSON 데이터를 조건 검색하거나 인덱싱해야 할 때
- 지리 데이터(PostGIS 확장)를 다룰 때
- SQL 표준을 엄격하게 따르는 환경
- SERIALIZABLE 격리가 진짜로 필요한 금융, 결제 시스템

**MySQL을 선택하는 경우**

- 단순 CRUD 중심의 웹 애플리케이션
- 읽기 처리량이 압도적으로 많은 서비스 (SNS 피드 등)
- 기존 레거시 코드나 인프라가 MySQL 기반일 때
- 관리 인력이 MySQL에 익숙한 환경

---

## 클라우드 매니지드 서비스

| 서비스 | PostgreSQL | MySQL |
|--------|-----------|-------|
| AWS | RDS for PostgreSQL, Aurora PostgreSQL | RDS for MySQL, Aurora MySQL |
| GCP | Cloud SQL for PostgreSQL, AlloyDB | Cloud SQL for MySQL |
| Azure | Azure Database for PostgreSQL | Azure Database for MySQL |

Aurora PostgreSQL과 Aurora MySQL은 AWS가 클라우드에 맞게 재설계한 엔진이라 기존 호환성을 유지하면서 성능을 높인 버전이다.

---

## 정리

PostgreSQL은 기능이 풍부하고 SQL 표준을 잘 따르며, 복잡한 쿼리와 다양한 데이터 타입이 필요한 시스템에 적합하다. MySQL은 단순 OLTP 워크로드와 대량 읽기 처리에서 가볍고 빠르다는 장점이 있다. 어떤 게 낫다기보다는 서비스의 쿼리 패턴과 팀의 경험을 기반으로 선택하는 게 현실적이다.
