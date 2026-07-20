---
layout: post
title: "[Daily morning study] 데이터베이스 커넥션 풀링(Connection Pooling) 개념과 동작 원리"
description: >
  #daily morning study
category: 
    - dms
    - dms-database
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 커넥션 풀링이란?

DB에 쿼리를 날리려면 먼저 TCP 연결을 맺고, 인증하고, 세션을 초기화하는 과정이 필요하다. 이 과정이 매 요청마다 반복되면 오버헤드가 커진다. 커넥션 풀링(Connection Pooling)은 이미 만들어 둔 커넥션을 재사용하는 방식으로 이 비용을 줄이는 기술이다.

```
요청 들어옴 → 풀에서 커넥션 꺼냄 → 쿼리 실행 → 커넥션 풀에 반납
             (새로 연결 X, 기존 것 재사용)
```

---

## 커넥션을 맺는 비용

PostgreSQL을 예시로, 새 커넥션을 만들 때 내부적으로 다음이 일어난다:

1. TCP 3-way handshake
2. TLS 핸드셰이크 (SSL 사용 시)
3. 서버 측 인증 처리
4. 백그라운드 워커 프로세스(worker process) 생성
5. 클라이언트 세션 초기화

실측상 로컬 환경에서도 1~5ms, 원격 서버에선 10~50ms 이상 걸릴 수 있다. 초당 수백 건 요청이 들어오는 서비스에서 매번 커넥션을 맺으면 DB 서버의 CPU와 메모리 소비도 급격히 늘어난다.

---

## 커넥션 풀의 동작 원리

```
┌─────────────────────────────────────────────┐
│               Connection Pool                │
│                                             │
│  [conn1: idle] [conn2: busy] [conn3: idle]  │
│                                             │
└─────────────────────────────────────────────┘
        ↑ 반납           ↓ 빌려줌
     스레드 A          스레드 B
```

1. **초기화**: 애플리케이션 시작 시 미리 정해진 수(`minimumIdle`)의 커넥션을 생성해 풀에 넣어둔다.
2. **대출(acquire)**: 스레드가 DB 작업을 할 때 풀에서 유휴(idle) 커넥션 하나를 가져간다.
3. **사용**: 가져간 커넥션으로 쿼리를 실행한다.
4. **반납(release)**: 작업이 끝나면 커넥션을 닫지 않고 풀에 돌려준다.
5. **대기**: 풀에 유휴 커넥션이 없으면 스레드는 `connectionTimeout` 시간만큼 기다린다. 시간 내에 반납이 없으면 예외를 던진다.

---

## 주요 풀링 라이브러리

| 언어/환경    | 대표 라이브러리                      |
|-------------|-------------------------------------|
| Java        | HikariCP, DBCP2, c3p0               |
| Python      | SQLAlchemy pool, psycopg2 pool      |
| Node.js     | pg-pool (node-postgres), knex       |
| Go          | database/sql 내장 풀                 |

Java 생태계에서는 **HikariCP**가 사실상 표준이다. Spring Boot 2.x 이상은 기본적으로 HikariCP를 사용한다.

---

## HikariCP 핵심 설정값

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10        # 풀의 최대 커넥션 수
      minimum-idle: 5              # 유지할 최소 유휴 커넥션 수
      connection-timeout: 30000    # 커넥션 대기 최대 시간 (ms)
      idle-timeout: 600000         # 유휴 커넥션 유지 시간 (ms)
      max-lifetime: 1800000        # 커넥션 최대 수명 (ms)
      keepalive-time: 30000        # 커넥션 살아있는지 확인 주기 (ms)
```

**maximum-pool-size**가 가장 중요한 값이다. 너무 작으면 대기 시간이 길어지고, 너무 크면 DB 서버에 과부하가 걸린다.

---

## 적절한 풀 사이즈 계산

HikariCP 공식 문서와 PostgreSQL 팀의 연구 결과에 따르면, 최적 풀 사이즈는 생각보다 작다.

```
풀 사이즈 = (코어 수 × 2) + 유효 디스크 스핀들 수
```

예를 들어 CPU 코어가 4개인 서버에서 SSD를 쓰면:

```
풀 사이즈 = (4 × 2) + 1 = 9
```

이 공식은 DB 서버 기준이다. 애플리케이션 서버가 여러 대라면 각 서버의 풀 사이즈 × 서버 대수가 DB가 받는 총 커넥션 수가 된다.

---

## 커넥션 고갈(Connection Exhaustion) 문제

풀 사이즈가 꽉 찼을 때 새 요청이 들어오면 `connectionTimeout`만큼 대기 후 예외가 발생한다.

```
HikariPool-1 - Connection is not available, request timed out after 30000ms
```

원인과 해결책:

| 원인 | 해결 방법 |
|------|----------|
| 슬로우 쿼리로 커넥션을 오래 점유 | 쿼리 최적화, 인덱스 추가 |
| `maximum-pool-size`가 너무 작음 | 풀 사이즈 증가 (단, DB 수용 한계 이내) |
| 트랜잭션 내에서 외부 API 호출 | 트랜잭션 범위 축소 |
| 커넥션 반납 누락 (커넥션 누수) | try-with-resources, 커넥션 누수 감지 설정 |

**커넥션 누수 감지** (HikariCP):

```yaml
hikari:
  leak-detection-threshold: 2000  # 2초 이상 반납 안 하면 경고 로그 출력
```

---

## 커넥션 유효성 검증

풀에서 꺼낸 커넥션이 이미 DB 서버 측에서 끊긴 상태일 수 있다. 이를 방지하기 위해 커넥션을 빌려줄 때 검증한다.

```yaml
hikari:
  connection-test-query: SELECT 1   # JDBC4 미지원 드라이버용
  # JDBC4 지원 드라이버(대부분)는 이 설정 없이도 자동 검증됨
```

`max-lifetime`을 DB 서버의 `wait_timeout`(MySQL) 또는 `tcp_keepalives_idle`(PostgreSQL) 보다 짧게 설정해야 유효하지 않은 커넥션이 풀에 남아있는 상황을 막을 수 있다.

---

## PgBouncer: DB 레벨 커넥션 풀러

애플리케이션 레벨 풀링 외에도 DB 앞단에 별도의 풀러를 두는 방식이 있다. PostgreSQL에서는 **PgBouncer**를 많이 쓴다.

```
앱 서버 (여러 대)
    ↓ 각자 connection pool
PgBouncer (중앙 집중 풀러)
    ↓ 실제 커넥션 (소수)
PostgreSQL 서버
```

PgBouncer의 세 가지 풀링 모드:

| 모드 | 설명 | 특징 |
|------|------|------|
| Session | 클라이언트 세션 당 커넥션 1개 유지 | 가장 안전 |
| Transaction | 트랜잭션 단위로 커넥션 할당 | 가장 많이 사용 |
| Statement | 쿼리 단위로 커넥션 할당 | prepared statement 제한 |

Transaction 모드에서는 SET, LISTEN, NOTIFY, advisory lock 등 세션 상태에 의존하는 기능을 쓰면 문제가 생길 수 있다.

---

## 모니터링 지표

커넥션 풀 상태를 추적할 때 봐야 하는 지표:

```
hikaricp_pending_threads    - 커넥션 대기 중인 스레드 수
hikaricp_active_connections - 현재 사용 중인 커넥션 수
hikaricp_idle_connections   - 유휴 커넥션 수
hikaricp_pool_size          - 현재 풀의 전체 커넥션 수
```

Prometheus + Grafana와 연동하면 `hikaricp_pending_threads`가 튀는 시점을 쉽게 파악할 수 있다. 이 값이 자주 0보다 크면 풀 사이즈 조정이나 쿼리 최적화가 필요하다는 신호다.

---

## 정리

- 커넥션 풀링은 매번 DB 연결을 맺는 오버헤드를 줄이기 위해 커넥션을 재사용하는 기술이다.
- 풀 사이즈는 크다고 좋은 게 아니라, DB 서버의 CPU 코어 수를 기준으로 계산한 적정값이 있다.
- 커넥션 고갈은 슬로우 쿼리, 트랜잭션 범위 문제, 커넥션 누수 등이 원인인 경우가 많다.
- 대규모 트래픽 환경에서는 애플리케이션 레벨 풀링에 더해 PgBouncer 같은 외부 풀러를 추가하는 것도 고려할 만하다.
