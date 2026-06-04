---
layout: post
title: "[Daily morning study] Redis Cluster 구성과 Failover"
description: >
  #daily morning study
category: 
    - dms
    - dms-database
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## Redis Cluster란

Redis Cluster는 데이터를 여러 노드에 자동으로 분산 저장하는 분산 Redis 구성이다. 단일 Redis 인스턴스의 한계인 메모리 용량과 처리량을 수평 확장으로 극복하면서, 일부 노드 장애에도 서비스가 계속되도록 고가용성을 제공한다.

핵심 특징:

- **자동 샤딩**: 데이터를 여러 노드에 분산
- **고가용성**: 일부 노드 장애 시 자동 Failover
- **선형 확장**: 노드 추가로 성능·용량 확장 가능

---

## 해시 슬롯 (Hash Slot)

Redis Cluster는 데이터를 **16384개의 해시 슬롯**으로 나눠서 관리한다. 각 마스터 노드가 슬롯의 일부 범위를 담당하는 방식이다.

키를 어느 슬롯에 저장할지 계산하는 공식:

```
slot = CRC16(key) % 16384
```

3개의 마스터 노드가 있다면 슬롯이 이렇게 나뉜다:

| 노드 | 담당 슬롯 범위 |
|------|--------------|
| Node A | 0 ~ 5460 |
| Node B | 5461 ~ 10922 |
| Node C | 10923 ~ 16383 |

클라이언트가 특정 키를 읽거나 쓰려 할 때, 해당 슬롯을 담당하는 노드에 요청을 보내야 한다. 잘못된 노드에 요청하면 `MOVED` 리다이렉션 응답이 온다.

```
MOVED 3999 127.0.0.1:6381
```

대부분의 Redis 클라이언트는 이 리다이렉션을 자동으로 처리한다.

### Hash Tags

`{}`로 묶은 부분만 해시 계산에 사용된다. 같은 슬롯에 저장해야 하는 관련 키들을 묶을 때 쓴다.

```
user:{1000}:profile
user:{1000}:session
```

두 키 모두 `1000`으로 슬롯을 계산하므로 같은 노드에 저장된다. 멀티 키 연산(`MGET`, `MSET` 등)은 모든 키가 같은 슬롯에 있어야 하므로 Hash Tags가 필수다.

---

## 클러스터 구조

Redis Cluster는 **마스터-레플리카** 구조로 이루어진다.

- **마스터 노드**: 실제 데이터 쓰기와 읽기를 처리하며 특정 슬롯 범위를 담당
- **레플리카 노드**: 마스터를 복제하며 장애 시 마스터로 승격 대기

권장 최소 구성은 마스터 3개 + 레플리카 3개(각 마스터당 1개)다.

```
Master A (슬롯 0~5460)       Replica A
Master B (슬롯 5461~10922)   Replica B
Master C (슬롯 10923~16383)  Replica C
```

마스터가 3개 미만이면 클러스터가 정상 동작하지 않는다. 쿼럼(과반수)을 구성하려면 최소 3개가 필요하기 때문이다.

---

## 클러스터 설정

`redis.conf`에 아래 항목을 추가하면 클러스터 모드가 활성화된다.

```conf
port 6379
cluster-enabled yes
cluster-config-file nodes-6379.conf
cluster-node-timeout 5000
appendonly yes
```

- `cluster-enabled yes`: 클러스터 모드 활성화
- `cluster-config-file`: 클러스터 상태를 저장하는 자동 관리 파일
- `cluster-node-timeout`: 노드를 장애로 판단하기까지 대기 시간(ms)

6개의 인스턴스를 띄운 후 `redis-cli`로 클러스터를 초기화한다.

```bash
redis-cli --cluster create \
  127.0.0.1:6379 127.0.0.1:6380 127.0.0.1:6381 \
  127.0.0.1:6382 127.0.0.1:6383 127.0.0.1:6384 \
  --cluster-replicas 1
```

`--cluster-replicas 1`은 마스터마다 레플리카 1개를 자동으로 배정하라는 뜻이다.

---

## Failover 과정

마스터 노드가 장애 상태가 되면 자동 Failover가 진행된다.

### 장애 감지

1. 노드들은 서로 **gossip 프로토콜**로 주기적으로 PING을 주고받는다
2. `cluster-node-timeout` 내에 PONG 응답이 없으면 해당 노드를 `PFAIL`(Possible Fail) 상태로 표시
3. 다른 노드들도 같은 노드를 PFAIL로 보고하면 `FAIL` 상태로 확정

### 레플리카 선거 (Election)

마스터가 FAIL 상태가 되면 그 마스터의 레플리카들이 새 마스터 선출을 시작한다.

1. 레플리카가 다른 마스터 노드들에게 `FAILOVER_AUTH_REQUEST`를 보냄
2. 마스터들이 `FAILOVER_AUTH_ACK`로 투표
3. **과반수 이상의 마스터 투표**를 받은 레플리카가 새 마스터로 승격
4. 새 마스터가 장애 마스터의 슬롯 범위를 인계받음

레플리카가 여러 개면, 복제 지연이 가장 적은(데이터가 가장 최신인) 레플리카가 우선적으로 선출된다.

```
Before:  Master A (정상) → Replica A
After:   Master A (장애) → Replica A (→ 새 Master A로 승격)
```

### 수동 Failover

운영 편의를 위해 수동으로 Failover를 트리거할 수 있다. 이 경우 데이터 손실 없이 안전하게 마스터를 교체할 수 있다.

```bash
# 레플리카 노드에서 실행
redis-cli -h 127.0.0.1 -p 6382 CLUSTER FAILOVER
```

---

## 주요 클러스터 명령어

```bash
# 클러스터 전체 상태 확인
CLUSTER INFO

# 모든 노드 목록과 슬롯 배치 확인
CLUSTER NODES

# 특정 키의 슬롯 번호 확인
CLUSTER KEYSLOT mykey

# 특정 슬롯에 속한 키 목록 확인
CLUSTER GETKEYSINSLOT 3999 10

# 노드를 클러스터에 추가
CLUSTER MEET <ip> <port>

# 현재 노드를 특정 마스터의 레플리카로 설정
CLUSTER REPLICATE <master-node-id>

# 클러스터 재구성 (슬롯 재배분 등)
redis-cli --cluster rebalance 127.0.0.1:6379
```

---

## 클러스터 제한사항

Redis Cluster를 도입할 때 알아둬야 할 제약이 있다.

**멀티 키 연산 제한**: `MGET`, `MSET`, `SUNION` 등 여러 키를 다루는 연산은 모든 키가 같은 슬롯에 있어야 한다. Hash Tags로 해결 가능하지만 설계 시 미리 고려해야 한다.

**데이터베이스 0만 사용**: 단일 Redis의 `SELECT` 명령어로 여러 DB를 사용하는 방식은 클러스터에서 지원하지 않는다. DB 0만 사용 가능하다.

**Lua 스크립트 제한**: 스크립트 내에서 접근하는 모든 키가 같은 슬롯에 있어야 한다.

**트랜잭션 제한**: `MULTI/EXEC` 트랜잭션도 같은 슬롯의 키만 묶을 수 있다.

---

## 클러스터 vs Sentinel

| | Redis Cluster | Redis Sentinel |
|--|--------------|---------------|
| 목적 | 수평 확장 + 고가용성 | 고가용성만 |
| 데이터 분산 | 자동 샤딩 | 단일 노드(전체 복제) |
| 스케일 아웃 | 가능 | 불가 |
| 설정 복잡도 | 높음 | 낮음 |
| 멀티 키 연산 | 제한적 | 자유로움 |
| 권장 상황 | 대용량 데이터, 높은 처리량 필요 | 중소규모, 단순 HA |

단순히 마스터 장애 시 자동 복구만 필요하다면 Sentinel이 간편하다. 데이터가 많아서 단일 노드에 담기 어렵거나 높은 처리량이 필요하다면 Cluster가 적합하다.
