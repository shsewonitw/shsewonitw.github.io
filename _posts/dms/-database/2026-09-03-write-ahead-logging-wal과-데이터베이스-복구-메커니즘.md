---
layout: post
title: "[Daily morning study] Write-Ahead Logging (WAL)과 데이터베이스 복구 메커니즘"
description: >
  #daily morning study
category: 
    - dms
    - dms-database
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## WAL이란?

WAL(Write-Ahead Logging)은 데이터베이스가 데이터를 디스크에 쓰기 전에 **반드시 먼저 로그를 기록**하는 원칙이다.

"Write-Ahead"라는 이름 그대로, 실제 데이터 변경보다 로그 작성이 항상 선행(ahead)된다.

이 원칙 덕분에 데이터베이스는 시스템이 갑자기 죽더라도 로그를 보고 복구할 수 있다.

---

## 왜 필요한가?

트랜잭션이 커밋되었는데 그 직후 서버가 크래시된다고 생각해보자.

- 메모리(버퍼 풀)에는 변경 사항이 있다.
- 디스크에는 아직 반영되지 않았다.
- 재시작하면 변경 내용이 사라진다. → **데이터 손실**

반대로, 커밋되지 않은 트랜잭션이 디스크에 부분적으로 쓰여진 상태에서 크래시가 발생하면?

- 일부만 쓰인 더러운 데이터가 디스크에 남는다. → **데이터 불일치**

WAL은 이 두 가지 문제를 모두 해결한다.

---

## WAL의 동작 방식

### 1. 트랜잭션 시작 시

트랜잭션이 시작되면 데이터베이스는 내부 트랜잭션 ID를 부여하고, 이후 모든 변경 작업을 로그에 기록할 준비를 한다.

### 2. 변경 발생 시

```
INSERT INTO orders (id, amount) VALUES (101, 5000);
```

이 쿼리가 실행되면 실제 테이블 페이지를 바꾸기 전에 WAL 버퍼에 다음과 같은 형태로 로그를 남긴다.

```
[LSN: 1234][TXN: 567][OP: INSERT][TABLE: orders][ROW: {id=101, amount=5000}]
```

- **LSN(Log Sequence Number):** 로그 레코드를 식별하는 단조 증가 번호다. 복구 시 어느 지점까지 처리했는지 파악하는 데 쓴다.
- **TXN:** 트랜잭션 ID
- **OP:** 수행한 작업 유형 (INSERT, UPDATE, DELETE, COMMIT, ROLLBACK 등)

### 3. 커밋 시

커밋이 호출되면 다음 순서로 진행된다.

```
1. WAL 버퍼 → WAL 로그 파일 (디스크) fsync
2. COMMIT 레코드를 WAL에 기록
3. 애플리케이션에 커밋 성공 응답 반환
4. 실제 데이터 페이지 변경은 나중에 비동기로 처리 (백그라운드 writer)
```

핵심은 **데이터 페이지보다 WAL이 먼저 디스크에 안전하게 저장**된다는 점이다.

---

## 체크포인트(Checkpoint)

WAL만으로는 로그가 무한히 쌓인다. 재시작 시 처음부터 전부 재생하면 너무 오래 걸린다.

체크포인트는 "이 시점까지는 메모리의 더티 페이지가 모두 디스크에 플러시되었다"는 마커다.

```
[CHECKPOINT at LSN: 5000]
```

복구 시에는 마지막 체크포인트 이후의 WAL 레코드만 재생하면 된다.

| 체크포인트 빈도 | 복구 시간 | I/O 부하 |
|---|---|---|
| 자주 (짧은 주기) | 짧다 | 높다 |
| 드물게 (긴 주기) | 길다 | 낮다 |

PostgreSQL에서는 `checkpoint_completion_target`, `checkpoint_timeout` 파라미터로 체크포인트 빈도를 조절한다.

---

## 크래시 복구 과정

데이터베이스가 재시작되면 3단계 복구 과정을 거친다.

### 1단계 — Analysis (분석)

WAL을 처음부터(또는 마지막 체크포인트부터) 스캔하여 크래시 당시 어떤 트랜잭션이 진행 중이었는지, 어떤 페이지가 더티했는지 파악한다.

### 2단계 — Redo (재실행)

마지막 체크포인트 이후의 WAL 레코드를 순서대로 재실행한다. 커밋된 트랜잭션이든 미완료 트랜잭션이든 일단 모두 적용한다.

```
Redo LSN 5001: INSERT orders ...
Redo LSN 5002: UPDATE users ...
Redo LSN 5003: COMMIT txn:567
...
```

### 3단계 — Undo (롤백)

Redo가 끝나면, 아직 COMMIT 레코드가 없는 미완료 트랜잭션들을 역순으로 롤백한다. 각 WAL 레코드에는 undo 정보도 포함되어 있어, 변경 전 상태로 되돌릴 수 있다.

```
Undo txn:789: DELETE FROM orders WHERE id=102
→ 해당 행을 다시 삽입
```

이 과정을 통해 **커밋된 트랜잭션은 보존, 미완료 트랜잭션은 제거**가 보장된다.

---

## WAL과 ACID

WAL은 ACID 속성과 직결된다.

| ACID 속성 | WAL의 역할 |
|---|---|
| Atomicity (원자성) | Undo를 통해 미완료 트랜잭션을 완전히 되돌림 |
| Durability (지속성) | 커밋 전 WAL을 fsync → 크래시 후에도 복구 가능 |

Consistency와 Isolation은 주로 트랜잭션 관리자와 잠금 메커니즘이 담당하지만, WAL이 원자성과 지속성을 보장함으로써 전체 ACID를 지탱한다.

---

## PostgreSQL에서의 WAL 구조

PostgreSQL은 `pg_wal/` 디렉터리에 WAL 세그먼트 파일을 저장한다.

```
$PGDATA/pg_wal/
  000000010000000000000001
  000000010000000000000002
  ...
```

각 파일은 기본 16MB이며 (`wal_segment_size`), LSN 순서로 쌓인다. 복제(Replication)에서는 스탠바이 서버가 이 WAL을 받아 재생함으로써 Primary와 동기화한다.

```sql
-- 현재 WAL 위치 확인
SELECT pg_current_wal_lsn();

-- WAL 생성 속도 확인
SELECT pg_walfile_name(pg_current_wal_lsn());
```

---

## InnoDB의 Redo Log

MySQL InnoDB도 동일한 WAL 원리를 사용하며, 이를 **Redo Log**라고 부른다.

```
ib_logfile0
ib_logfile1
```

Redo Log는 고정 크기의 링 버퍼 구조다. 꽉 차면 처음으로 돌아가 덮어쓴다. 이 때문에 로그가 너무 빨리 회전하면 체크포인트가 강제로 발생해 I/O 스파이크가 생긴다.

MySQL 8.0부터는 Redo Log 크기를 동적으로 조절할 수 있게 되었다.

```sql
-- InnoDB Redo Log 상태 확인
SHOW ENGINE INNODB STATUS\G
```

---

## WAL과 성능 트레이드오프

WAL이 안전성을 보장하지만 성능 비용이 따른다.

**fsync의 비용**

커밋마다 WAL을 디스크에 강제 플러시(fsync)하면 I/O 비용이 크다. 이를 줄이기 위한 설정들이 있다.

PostgreSQL:

```
synchronous_commit = on      # 커밋마다 fsync (기본값, 가장 안전)
synchronous_commit = off     # fsync 생략 → 최대 600ms 데이터 손실 가능
synchronous_commit = local   # 로컬은 동기, 복제는 비동기
```

`synchronous_commit = off`로 설정하면 트랜잭션 처리량이 크게 높아지지만, 크래시 시 최근 커밋 몇 건이 사라질 수 있다. 로그성 데이터처럼 손실 허용 가능한 워크로드에 적합하다.

**그룹 커밋(Group Commit)**

여러 트랜잭션의 WAL을 모아 한 번에 fsync한다. 개별 fsync 횟수를 줄여 I/O 효율을 높인다. PostgreSQL과 InnoDB 모두 기본적으로 그룹 커밋을 지원한다.

---

## 요약

WAL은 데이터베이스의 신뢰성 기반이다.

- **원칙:** 실제 데이터보다 로그를 먼저 디스크에 기록한다.
- **복구 흐름:** Analysis → Redo → Undo
- **체크포인트:** 복구 범위를 제한해 재시작 시간을 단축한다.
- **트레이드오프:** 안전성(fsync)과 성능(group commit, synchronous_commit) 사이의 균형이 중요하다.

데이터베이스 복제, MVCC, 트랜잭션 격리 수준 모두 WAL 위에서 동작한다. 내부 동작을 이해하면 성능 튜닝과 장애 대응이 훨씬 쉬워진다.
