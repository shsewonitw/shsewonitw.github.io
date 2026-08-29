---
layout: post
title: "[Daily morning study] 리눅스 CFS 스케줄러 (Completely Fair Scheduler)"
description: >
  #daily morning study
category: 
    - dms
    - dms-os
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## CFS란?

CFS(Completely Fair Scheduler)는 리눅스 커널 2.6.23(2007년)부터 기본 CPU 스케줄러로 채택된 알고리즘이다. 기존의 O(1) 스케줄러를 대체했으며, 목표는 이름 그대로 "완전히 공정한" CPU 시간 분배다.

전통적인 Priority Queue 기반 스케줄러는 우선순위가 높은 프로세스가 계속 실행되면 낮은 우선순위 프로세스가 굶는(starvation) 문제가 있었다. CFS는 가상 시계(virtual clock)를 도입해 이를 해결했다.

---

## 핵심 개념: vruntime

CFS의 핵심은 **vruntime(virtual runtime)**이다. 각 프로세스가 CPU를 얼마나 사용했는지를 나타내는 누적 값으로, 실제 실행 시간을 가중치로 정규화한 값이다.

```
vruntime += 실제 실행 시간 × (기준 가중치 / 프로세스 가중치)
```

- 가중치가 높은(nice 값이 낮은) 프로세스: vruntime이 느리게 증가 → 더 자주 선택됨
- 가중치가 낮은(nice 값이 높은) 프로세스: vruntime이 빠르게 증가 → 덜 자주 선택됨

CFS는 항상 **vruntime이 가장 작은 프로세스**를 다음 실행 대상으로 선택한다.

---

## nice 값과 가중치

리눅스에서 프로세스 우선순위는 `nice` 값으로 조정한다. nice 범위는 `-20`(최고 우선순위) ~ `+19`(최저 우선순위)다.

CFS는 nice 값을 내부 가중치로 변환해 사용한다.

| nice 값 | 가중치 (weight) | 비고 |
|--------|----------------|------|
| -20    | 88761          | 최고 우선순위 |
| 0      | 1024           | 기본값 |
| +19    | 15             | 최저 우선순위 |

nice 0 기준으로 nice 값이 1 증가할 때마다 CPU 점유율이 약 10% 감소하도록 설계되어 있다.

---

## Red-Black Tree (RB 트리)

CFS는 실행 가능한 프로세스들을 **Red-Black Tree**에 vruntime 기준으로 정렬해서 관리한다.

```
         [vruntime=10]
        /              \
  [vruntime=5]    [vruntime=15]
      /
[vruntime=3]
```

- **가장 왼쪽 노드**: vruntime이 가장 작은 프로세스 → 다음 실행 대상
- 삽입/삭제: O(log N)
- 최솟값 탐색: O(1) (커널이 캐시해 둠)

프로세스가 CPU를 양보하면 vruntime이 갱신된 뒤 트리에 재삽입된다. 새로운 프로세스는 현재 트리의 최솟값에 맞춰 vruntime이 초기화되므로 오래된 프로세스들을 굶기지 않는다.

---

## 스케줄링 레이턴시와 타임슬라이스

CFS는 고정 타임슬라이스 대신 **스케줄링 레이턴시(sched_latency)**를 기반으로 동적으로 타임슬라이스를 결정한다.

```
타임슬라이스 = (sched_latency / 실행 가능 프로세스 수) × (프로세스 가중치 / 총 가중치)
```

- 기본 `sched_latency`: 6ms ~ 48ms (커널 설정에 따라 다름)
- 프로세스 수가 많아지면 타임슬라이스가 줄어들어 반응성이 유지됨
- 너무 작아지지 않도록 `sched_min_granularity`(최소 타임슬라이스)로 하한을 설정

```bash
# 스케줄러 파라미터 확인
cat /proc/sys/kernel/sched_latency_ns        # 스케줄링 레이턴시 (나노초)
cat /proc/sys/kernel/sched_min_granularity_ns # 최소 타임슬라이스
cat /proc/sys/kernel/sched_wakeup_granularity_ns
```

---

## 슬립 프로세스 처리

I/O 대기나 sleep 상태였던 프로세스가 깨어나면, vruntime이 오래된 값일 수 있다. 이를 그대로 사용하면 CPU를 독점하게 되므로 CFS는 깨어난 프로세스의 vruntime을 보정한다.

```
vruntime = max(프로세스의 기존 vruntime, 최솟값 - sched_latency)
```

덕분에 방금 깨어난 프로세스가 즉시 실행 기회를 얻지만, 과도하게 독점하지도 않는다. 이 보정이 인터랙티브 애플리케이션의 반응성을 높이는 핵심이다.

---

## CFS 스케줄러 클래스와 우선순위

리눅스는 단일 스케줄러가 아닌 **스케줄러 클래스(scheduler class)** 계층으로 구성된다.

```
Stop     (priority 1) — 마이그레이션 스레드 등 특수 목적
Deadline (priority 2) — SCHED_DEADLINE: 실시간 마감 스케줄링
RT       (priority 3) — SCHED_FIFO, SCHED_RR: 실시간 프로세스
Fair     (priority 4) — SCHED_NORMAL, SCHED_BATCH: CFS
Idle     (priority 5) — SCHED_IDLE: CPU가 유휴일 때만 실행
```

RT 클래스 프로세스가 존재하면 CFS(Fair) 클래스 프로세스는 실행되지 않는다. 일반적인 유저 프로세스는 Fair 클래스에 속한다.

```c
// 프로세스 스케줄링 정책 설정 예시 (C)
struct sched_param param = { .sched_priority = 50 };
sched_setscheduler(pid, SCHED_FIFO, &param);  // RT 클래스
sched_setscheduler(pid, SCHED_NORMAL, &param); // CFS 클래스
```

---

## 멀티코어와 로드 밸런싱

CFS는 멀티코어 환경에서 **런큐(run queue)**를 CPU마다 별도로 관리하고, 주기적으로 로드 밸런싱을 수행한다.

- 각 CPU 코어가 독립적인 RB 트리를 가짐
- 실행 가능한 프로세스가 없는 코어가 바쁜 코어에서 프로세스를 빼앗아 옴 (work stealing)
- NUMA(Non-Uniform Memory Access) 구조를 고려해 메모리 지역성도 최적화

```bash
# CPU별 런큐 상태 확인
cat /proc/schedstat
```

---

## cgroup과 CFS 대역폭 제어

컨테이너 환경에서 CPU 제한은 cgroup의 CFS 대역폭 제어(bandwidth control)로 구현된다.

```bash
# cgroup v2: CPU 할당량 설정
echo "100000 200000" > /sys/fs/cgroup/mygroup/cpu.max
# 200ms 주기에서 최대 100ms 사용 → CPU 50% 상한
```

Docker나 Kubernetes의 `--cpus`, `resources.limits.cpu` 설정이 내부적으로 이 메커니즘을 사용한다.

```yaml
# Kubernetes Pod CPU 제한
resources:
  requests:
    cpu: "500m"   # 0.5 CPU 보장 (nice값/가중치로 구현)
  limits:
    cpu: "1"      # 1 CPU 상한 (CFS bandwidth throttle로 구현)
```

---

## 정리

| 항목 | 내용 |
|------|------|
| 핵심 자료구조 | Red-Black Tree (vruntime 기준 정렬) |
| 다음 실행 프로세스 | vruntime이 가장 작은 프로세스 |
| 우선순위 표현 | nice 값 → 가중치 변환 |
| 타임슬라이스 | 동적 결정 (sched_latency 기반) |
| 공정성 보장 | vruntime 정규화 + 슬립 보정 |
| 멀티코어 | CPU별 런큐 + work stealing |
| 컨테이너 연동 | cgroup CFS bandwidth control |

CFS는 단순한 라운드 로빈 대신 vruntime이라는 추상화된 "공정성 척도"를 도입해서, 우선순위와 공정성을 동시에 달성한 스케줄러다. 컨테이너 환경이 일반화된 지금도 쿠버네티스의 CPU 요청/제한은 결국 CFS 위에서 동작한다.
