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

## 왜 둘을 비교하는가

오픈소스 RDBMS 시장에서 가장 많이 쓰이는 두 DB가 MySQL과 PostgreSQL이다. MySQL은 웹 서비스 초창기부터 LAMP 스택의 한 축을 담당했고, PostgreSQL(이하 Postgres)은 "가장 기능이 풍부한 오픈소스 DB"를 목표로 발전해왔다. 무엇을 선택하느냐는 단순한 취향 문제가 아니라 트랜잭션 동시성, 데이터 타입 요구사항, 확장성, 라이선스 등 실질적인 차이에서 비롯된다.

---

## 아키텍처 차이

### MySQL의 스토리지 엔진 구조

MySQL은 **플러그인 가능한 스토리지 엔진** 구조를 갖는다. 쿼리 레이어와 스토리지 레이어가 분리되어 있어, 같은 SQL로 InnoDB, MyISAM, Memory, CSV 등 다른 엔진을 사용할 수 있다.

- **InnoDB** — 기본 엔진. 트랜잭션, 외래키, Row-level locking 지원
- **MyISAM** — 트랜잭션 없음, Table-level locking. 읽기 집약적 레거시에 사용
- 현대 MySQL에서는 거의 InnoDB만 쓴다고 봐도 무방하다

### PostgreSQL의 단일 스토리지 엔진

Postgres는 스토리지 엔진이 하나다. 대신 **테이블 액세스 메서드(Table Access Method) API**를 통해 외부 엔진을 붙일 수 있는 구조(pg17 이후 확장 예정)가 있지만 실용적으로는 기본 heap 스토리지를 사용한다. 복잡성 대신 심층적인 기능 통합에 집중했다.

---

## MVCC 구현 방식

두 DB 모두 **MVCC(Multi-Version Concurrency Control)**를 사용하지만 방식이 다르다.

### MySQL InnoDB의 MVCC

Undo 로그 기반. 행을 업데이트하면 **이전 버전을 undo 로그에** 보관하고 원본 행을 새 값으로 덮어쓴다. 오래된 버전이 필요한 트랜잭션은 undo 로그를 역추적한다.

```
행 업데이트 전: [ data=old | txn_id=100 | roll_pointer → undo_log ]
행 업데이트 후: [ data=new | txn_id=200 | roll_pointer → old_version ]
                                               ↓
                                           undo_log: old_version 보관
```

장기 트랜잭션이 있으면 undo 로그가 쌓여 **undo tablespace** 부담이 커진다.

### PostgreSQL의 MVCC

Postgres는 **행 자체에 여러 버전**을 테이블 내부에 저장한다 (Heap tuple versioning). 업데이트하면 새 튜플을 테이블에 INSERT하고 이전 튜플의 `xmax`를 설정해 삭제 표시한다.

```
업데이트 전:  [ xmin=100, xmax=0,   data=old ]  ← 살아있는 튜플
업데이트 후:  [ xmin=100, xmax=200, data=old ]  ← 죽은 튜플 (xmax 설정됨)
              [ xmin=200, xmax=0,   data=new ]  ← 새 튜플
```

죽은 튜플은 **VACUUM** 프로세스가 나중에 회수한다. VACUUM이 제때 돌지 않으면 테이블 bloat이 생긴다.

| 항목 | MySQL InnoDB | PostgreSQL |
|------|-------------|-----------|
| 구버전 보관 위치 | Undo 로그 | 테이블 내부 (dead tuple) |
| 정리 메커니즘 | Purge thread | VACUUM |
| 장기 트랜잭션 영향 | undo log 증가 | table bloat |

---

## 트랜잭션 격리 수준

| 격리 수준 | MySQL InnoDB | PostgreSQL |
|----------|-------------|-----------|
| READ UNCOMMITTED | 지원 | 없음 (READ COMMITTED로 처리) |
| READ COMMITTED | 지원 | 지원 |
| REPEATABLE READ | **기본값**, Gap Lock으로 Phantom Read 방지 | 지원 |
| SERIALIZABLE | 지원 | 지원 (SSI 구현) |

Postgres의 **SSI(Serializable Snapshot Isolation)**는 잠금 없이 직렬화 가능성을 보장하는 기법으로, 읽기가 많은 OLTP에서 성능 이점이 있다.

MySQL의 REPEATABLE READ에서 Gap Lock은 트랜잭션 간 교착(Deadlock)을 유발하기 쉽다. 고도로 동시적인 쓰기 워크로드에서 잠금 경합 문제가 자주 발생한다.

---

## 데이터 타입과 확장성

PostgreSQL이 제공하는 데이터 타입이 훨씬 풍부하다.

| 타입 | MySQL | PostgreSQL |
|------|-------|-----------|
| JSON | JSON, JSON_TABLE (8.0+) | JSON, JSONB (인덱싱 가능) |
| 배열 | 없음 | `integer[]`, `text[]` 등 기본 지원 |
| 범위 | 없음 | `int4range`, `tsrange` 등 |
| UUID | VARCHAR로 저장 | 전용 UUID 타입 |
| 기하/공간 | 기본 Geometry | PostGIS 확장으로 정밀 GIS |
| 전문 검색 | FULLTEXT 인덱스 | `tsvector`, `tsquery`, GIN 인덱스 |
| 사용자 정의 타입 | ENUM, SET 정도 | `CREATE TYPE`, 복합 타입, 도메인 |

Postgres에서 JSONB는 바이너리 포맷으로 저장되어 파싱 없이 인덱싱할 수 있다. MongoDB 대신 PostgreSQL로 JSON 중심 서비스를 구축하는 사례가 많은 이유가 여기에 있다.

---

## 인덱스 종류

PostgreSQL은 인덱스 선택지가 다양하다.

| 인덱스 타입 | MySQL | PostgreSQL |
|-----------|-------|-----------|
| B-Tree | O | O |
| Hash | O (제한적) | O |
| GiST | X | O (기하, 범위, 전문검색) |
| GIN | X | O (배열, JSONB, 전문검색) |
| BRIN | X | O (물리적으로 정렬된 대용량 테이블) |
| 부분 인덱스 | X | O (`WHERE` 조건부 인덱스) |
| 표현식 인덱스 | 함수 인덱스(제한적) | O (`lower(email)` 등) |

부분 인덱스(Partial Index) 예시:

```sql
-- 활성 사용자에 대한 인덱스만 생성
CREATE INDEX idx_active_users ON users (email)
WHERE is_active = TRUE;
```

비활성 사용자가 많다면 인덱스 크기를 크게 줄일 수 있다.

---

## 복제와 고가용성

### MySQL 복제

- **Binary Log 기반 복제**: Statement, Row, Mixed 세 가지 포맷
- **GTID(Global Transaction ID)** 복제: 5.6부터 도입, 페일오버가 쉬워짐
- **Group Replication / InnoDB Cluster**: MySQL 8.0의 공식 HA 솔루션
- **ProxySQL** 같은 미들웨어로 읽기/쓰기 분리를 많이 구성

### PostgreSQL 복제

- **WAL 기반 Streaming Replication**: 기본 HA 방식. `primary_conninfo`로 스탠바이 구성
- **Logical Replication**: 특정 테이블/논리 변경사항만 복제. 이기종 버전 간 복제 가능
- **Patroni**: 업계 표준 HA 도구. etcd/Consul/ZooKeeper로 리더 선출 관리

```
# Patroni 구성 예시 (patroni.yml 일부)
restapi:
  listen: 0.0.0.0:8008
  connect_address: 10.0.0.1:8008

postgresql:
  listen: 0.0.0.0:5432
  data_dir: /var/lib/postgresql/14/main
  parameters:
    max_connections: 200
    wal_level: replica
```

---

## 성능 특성

### 읽기 성능

단순 PK 조회나 인덱스 범위 스캔은 MySQL이 약간 빠른 경우가 많다. InnoDB의 **Clustered Index** 구조(PK가 테이블 데이터와 함께 저장)가 PK 기반 조회에 최적화되어 있기 때문이다.

Postgres는 테이블과 인덱스가 분리된 **Heap 구조**다. PK로 조회해도 인덱스 → heap 테이블 순으로 두 번 접근(Index Scan + Heap Fetch)하는 경우가 많다. 다만 `Index Only Scan`이나 Covering Index로 이를 회피할 수 있다.

### 쓰기 성능

복잡한 쿼리나 JOIN이 많은 OLAP성 쿼리에서는 Postgres의 **쿼리 플래너**가 더 정교한 실행 계획을 만들어내는 경우가 많다. Hash Join, Merge Join, 병렬 쿼리 실행이 더 풍부하게 지원된다.

### VACUUM 오버헤드

Postgres의 VACUUM은 dead tuple을 회수하는 필수 작업이다. Autovacuum 설정이 잘못되면 테이블 bloat이 발생해 성능이 저하된다. write-heavy 환경에서는 VACUUM 튜닝이 중요한 운영 포인트다.

```sql
-- 테이블 bloat 확인
SELECT relname, n_dead_tup, n_live_tup, 
       round(n_dead_tup::numeric / nullif(n_live_tup,0) * 100, 2) AS dead_ratio
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

---

## 스토어드 프로시저 / 함수

Postgres는 여러 언어로 프로시저를 작성할 수 있다.

| 항목 | MySQL | PostgreSQL |
|------|-------|-----------|
| 프로시저 언어 | SQL, 제한적 | PL/pgSQL, PL/Python, PL/Perl, PL/v8 등 |
| 반환 타입 | 단일 결과셋 | TABLE, SETOF, 복합 타입 등 |
| 트랜잭션 제어 | 프로시저 내 BEGIN/COMMIT (8.0+) | 동일 |

PL/pgSQL 예시:

```sql
CREATE OR REPLACE FUNCTION get_user_rank(p_user_id BIGINT)
RETURNS INT AS $$
DECLARE
  v_rank INT;
BEGIN
  SELECT rank INTO v_rank
  FROM (
    SELECT user_id, RANK() OVER (ORDER BY score DESC) AS rank
    FROM users
  ) ranked
  WHERE user_id = p_user_id;
  RETURN v_rank;
END;
$$ LANGUAGE plpgsql;
```

---

## 라이선스와 생태계

| 항목 | MySQL | PostgreSQL |
|------|-------|-----------|
| 라이선스 | GPL v2 (오픈소스), 상용 라이선스 별도 | PostgreSQL License (BSD 유사, 매우 자유로움) |
| 소유 | Oracle 인수 (2010) | 커뮤니티 주도 |
| 클라우드 관리형 | AWS RDS/Aurora, GCP Cloud SQL, Azure DB | RDS for PostgreSQL, Cloud SQL, Azure DB |
| ORM 지원 | 매우 광범위 | 광범위 (점점 Postgres 우선 추세) |

MySQL은 Oracle 인수 이후 일부 개발자들이 MariaDB로 분기했다. 라이선스 리스크를 우려하는 기업은 Postgres를 선호하는 경향이 있다.

---

## 언제 무엇을 선택할까

**MySQL이 유리한 경우**
- 단순 CRUD 중심의 읽기 집약적 웹 서비스
- MySQL 생태계에 익숙한 팀
- 기존 MySQL 인프라와의 연동

**PostgreSQL이 유리한 경우**
- 복잡한 쿼리, 분석, 집계가 많은 서비스
- JSONB, 배열, 범위 타입 등 다양한 데이터 타입이 필요한 경우
- GIS, 전문 검색 등 확장 기능이 필요한 경우
- 엄격한 직렬화 격리가 필요한 금융/핀테크 도메인
- 장기적으로 라이선스 자유도가 중요한 경우

---

## 정리

| 항목 | MySQL | PostgreSQL |
|------|-------|-----------|
| MVCC 방식 | Undo 로그 | Heap tuple versioning + VACUUM |
| 기본 격리 수준 | REPEATABLE READ | READ COMMITTED |
| 데이터 타입 | 보통 | 매우 풍부 (JSONB, 배열, GIS 등) |
| 인덱스 종류 | B-Tree 중심 | B-Tree, GIN, GiST, BRIN 등 |
| 단순 읽기 | 빠름 (Clustered PK) | 보통 (Heap 분리) |
| 복잡한 쿼리 | 보통 | 우수한 쿼리 플래너 |
| 확장성 | 제한적 | 매우 풍부 (Extension 시스템) |
| 라이선스 | GPL / Oracle | BSD 유사, 매우 자유로움 |

둘 다 성숙한 프로덕션 DB지만, 기능의 깊이와 확장성 측면에서 현대 서비스는 점점 PostgreSQL을 선택하는 추세다. MySQL은 단순한 웹 서비스에서 여전히 충분한 선택지이고, 운영 노하우가 풍부한 팀이라면 속도 면에서 이점을 가져갈 수 있다.
