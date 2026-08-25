---
layout: post
title: "[Daily morning study] 샤딩(Sharding)과 파티셔닝 전략"
description: >
  #daily morning study
category: 
    - dms
    - dms-database
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 왜 샤딩과 파티셔닝이 필요한가

데이터가 수억 건을 넘어가면 단일 데이터베이스 서버로는 버티기 어려워진다. 쿼리 응답 시간이 늘어나고, 디스크 I/O가 병목이 되고, 장애 발생 시 전체 서비스가 멈춘다. 이 문제를 해결하는 대표적인 두 가지 접근이 **파티셔닝(Partitioning)** 과 **샤딩(Sharding)** 이다.

두 개념은 자주 혼동되는데, 핵심 차이는 **어디서 분할하느냐** 에 있다.

| 구분 | 파티셔닝 | 샤딩 |
|------|----------|------|
| 분할 위치 | 단일 DB 서버 내부 | 여러 DB 서버로 분산 |
| 스케일 방식 | 논리적 분할 (Scale-Up 지원) | 수평 분산 (Scale-Out) |
| 투명성 | 애플리케이션에 투명 | 애플리케이션이 인식 필요 |
| 관리 복잡도 | 낮음 | 높음 |

---

## 파티셔닝(Partitioning)

하나의 테이블을 물리적으로 여러 조각(파티션)으로 나누되, 논리적으로는 하나의 테이블처럼 동작하게 만드는 기법이다. 데이터베이스 엔진이 파티션을 인식하고, 쿼리 옵티마이저가 필요한 파티션만 읽는 **파티션 프루닝(Partition Pruning)** 을 수행한다.

### 수평 파티셔닝 (Horizontal Partitioning)

행(Row)을 기준으로 분할한다. 같은 스키마를 가진 여러 파티션에 데이터를 나눠 저장한다.

```sql
-- PostgreSQL 범위 파티셔닝 예시
CREATE TABLE orders (
    id          BIGINT,
    created_at  DATE,
    user_id     BIGINT,
    amount      NUMERIC
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2024 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

CREATE TABLE orders_2025 PARTITION OF orders
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

CREATE TABLE orders_2026 PARTITION OF orders
    FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');
```

위처럼 설정하면 `WHERE created_at BETWEEN '2026-01-01' AND '2026-12-31'` 쿼리는 `orders_2026` 파티션만 스캔한다.

### 수직 파티셔닝 (Vertical Partitioning)

열(Column)을 기준으로 분할한다. 자주 조회되는 컬럼과 드물게 조회되는 컬럼(예: BLOB, TEXT)을 다른 테이블로 분리한다.

```
users 테이블
  → users_core   (id, name, email, created_at)  ← 자주 조회
  → users_profile (id, bio, avatar_blob, metadata) ← 드물게 조회
```

자주 조회되지 않는 대용량 컬럼이 메모리와 I/O를 낭비하는 걸 막을 수 있다.

### 파티셔닝 전략 종류

**범위 파티셔닝 (Range)**: 날짜, 숫자 범위를 기준으로 분할. 시계열 데이터, 로그 테이블에 적합하다.

**리스트 파티셔닝 (List)**: 특정 값 목록으로 분할. 지역 코드, 카테고리처럼 열거 가능한 값에 사용한다.

```sql
CREATE TABLE sales PARTITION BY LIST (region);

CREATE TABLE sales_korea PARTITION OF sales
    FOR VALUES IN ('KR', 'SEOUL', 'BUSAN');

CREATE TABLE sales_usa PARTITION OF sales
    FOR VALUES IN ('US', 'NY', 'LA');
```

**해시 파티셔닝 (Hash)**: 파티션 키에 해시 함수를 적용해 분할. 균등 분산이 목적일 때 사용한다.

```sql
CREATE TABLE user_events PARTITION BY HASH (user_id);

CREATE TABLE user_events_0 PARTITION OF user_events
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);
-- ... 0~3까지 4개 파티션
```

---

## 샤딩(Sharding)

샤딩은 데이터를 여러 **물리적으로 독립된 데이터베이스 서버** 에 분산하는 기법이다. 각 서버를 **샤드(Shard)** 라 부르고, 각 샤드는 전체 데이터의 일부만 담당한다.

### 샤드 키(Shard Key) 선택

샤딩 설계에서 가장 중요한 결정이다. 샤드 키는 어떤 데이터를 어느 샤드에 저장할지 결정한다.

좋은 샤드 키의 조건:
- **높은 카디널리티(Cardinality)**: 값이 다양해야 균등 분산이 된다.
- **균등 분포**: 특정 샤드에 데이터가 쏠리는 **핫스팟(Hotspot)** 이 없어야 한다.
- **쿼리 친화성**: 자주 쓰는 쿼리가 단일 샤드에서 해결되어야 한다.

### 샤딩 전략 종류

**범위 기반 샤딩 (Range-based Sharding)**

```
user_id 1 ~ 1,000,000   → Shard 1
user_id 1,000,001 ~ 2,000,000 → Shard 2
user_id 2,000,001 ~     → Shard 3
```

구현이 단순하고 범위 쿼리에 유리하다. 다만 신규 데이터가 최신 샤드에 몰리는 핫스팟 문제가 생기기 쉽다.

**해시 기반 샤딩 (Hash-based Sharding)**

```python
shard_id = hash(user_id) % num_shards

# 예시
user_id = 12345
shard_id = 12345 % 4  # → shard 1
```

균등 분산에 유리하다. 단, 샤드 수를 늘리면 기존 해시 값이 전부 바뀌어 대규모 데이터 재배치가 필요하다. 이를 완화하기 위해 **컨시스턴트 해싱(Consistent Hashing)** 을 쓴다.

**디렉토리 기반 샤딩 (Directory-based Sharding)**

별도의 룩업 테이블(디렉토리)이 샤드 위치를 기록한다.

```
샤드 디렉토리 DB:
user_id 1~500  → shard_A
user_id 501~700 → shard_B
...
```

유연한 샤드 재배치가 가능하지만, 디렉토리 DB가 단일 장애점(SPOF)이 될 수 있다.

### 컨시스턴트 해싱 (Consistent Hashing)

샤드 수가 변할 때 재배치되는 데이터를 최소화하기 위해 사용한다. 해시 공간을 원형(Ring)으로 구성하고, 샤드와 데이터를 같은 링 위에 배치한다.

```
Ring:   0 ─── A ─── B ─── C ─── 0 (원형)

데이터 key의 해시값이 A와 B 사이 → Shard B에 저장
            B와 C 사이 → Shard C에 저장
```

샤드 C를 추가하면 C 이전 구간의 데이터 일부만 재배치된다. 기존 해시 방식에서 전체를 재배치해야 했던 것과 비교하면 훨씬 효율적이다.

---

## 샤딩의 주요 문제점

### 교차 샤드 쿼리 (Cross-shard Query)

JOIN이나 집계(COUNT, SUM)를 여러 샤드에 걸쳐 해야 할 때 복잡해진다. 각 샤드에서 부분 결과를 가져와 애플리케이션 레이어에서 합쳐야 하는 경우가 많다.

```python
# 모든 샤드에서 집계
totals = []
for shard in shards:
    result = shard.execute("SELECT SUM(amount) FROM orders WHERE date = '2026-08-25'")
    totals.append(result)

global_total = sum(totals)  # 애플리케이션에서 합산
```

### 분산 트랜잭션

여러 샤드에 걸친 트랜잭션은 ACID 보장이 어렵다. 2PC(Two-Phase Commit)를 쓸 수 있지만 성능 오버헤드가 크고, 대부분의 시스템은 샤드 설계 자체로 교차 트랜잭션이 거의 없도록 만든다.

### 재샤딩 (Resharding)

샤드를 추가하거나 줄일 때 기존 데이터를 재분배해야 한다. 이 과정에서 서비스 중단이나 성능 저하가 발생할 수 있어 사전 계획이 중요하다.

### 데이터 불균형 (Hotspot)

특정 샤드에 트래픽이 집중되는 현상이다. 예를 들어 인플루언서의 user_id가 특정 샤드에 집중되면 그 샤드만 과부하가 걸린다. 이를 해결하려면 핫 샤드를 더 작은 단위로 분할하거나, 샤드 키를 복합 키로 변경한다.

---

## 파티셔닝 vs 샤딩 선택 기준

| 상황 | 권장 방식 |
|------|-----------|
| 단일 서버로 처리 가능하지만 테이블이 너무 큰 경우 | 파티셔닝 |
| 읽기/쓰기 부하가 단일 서버 한계를 초과 | 샤딩 |
| 시계열 데이터로 오래된 데이터를 주기적으로 아카이브 | 범위 파티셔닝 |
| 글로벌 서비스로 지역별 데이터 격리가 필요 | 리스트 파티셔닝 또는 지역 샤딩 |
| 데이터가 수백 TB를 넘고 트래픽도 매우 높음 | 파티셔닝 + 샤딩 병행 |

---

## 실제 사용 사례

**Instagram**: user_id 기반 해시 샤딩. 같은 사용자의 사진, 팔로우, 피드 데이터를 같은 샤드에 배치해 교차 샤드 쿼리를 최소화한다.

**Vitess (YouTube/YouTube 채택)**: MySQL 클러스터 위에서 투명한 샤딩을 제공한다. 애플리케이션은 단일 MySQL처럼 쿼리하고, Vitess가 샤드 라우팅을 처리한다.

**MongoDB**: 자체 샤딩 기능(Config Server + mongos 라우터)을 내장하고 있어, 컬렉션 단위로 샤드 키를 설정할 수 있다.

---

## 정리

- **파티셔닝**은 단일 서버 안에서 테이블을 논리적으로 나눠 쿼리 성능을 높이는 기법이다.
- **샤딩**은 여러 서버로 데이터를 수평 분산해 처리 용량 자체를 늘리는 기법이다.
- 샤드 키 선택이 샤딩 설계의 핵심이며, 잘못 선택하면 핫스팟과 교차 샤드 쿼리 문제가 발생한다.
- 컨시스턴트 해싱은 샤드 추가 시 재배치 비용을 최소화하는 데 효과적이다.
- 대부분의 서비스는 먼저 파티셔닝으로 성능을 확보하고, 한계에 도달했을 때 샤딩을 도입한다.
