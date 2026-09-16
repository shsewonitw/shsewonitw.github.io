---
layout: post
title: "[Daily morning study] Linux OOM Killer 동작 원리와 메모리 부족 상황 처리"
description: >
  #daily morning study
category: 
    - dms
    - dms-os
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## OOM Killer란

OOM(Out-Of-Memory) Killer는 리눅스 커널의 메모리 관리 메커니즘이다. 물리 메모리와 스왑 공간이 모두 소진되어 정상적인 메모리 할당이 불가능해지면, 커널은 특정 프로세스를 강제 종료해 메모리를 확보하고 시스템 전체가 멈추는 상황을 방지한다.

OOM Killer는 시스템의 마지막 방어선이다. OOM이 자주 발생한다면 그건 애플리케이션의 메모리 누수이거나, 서버 사양이 워크로드에 맞지 않는다는 신호다.

---

## 리눅스의 메모리 오버커밋

리눅스는 기본적으로 **메모리 오버커밋(overcommit)**을 허용한다. 프로세스가 `malloc()`을 호출할 때 실제 물리 메모리를 즉시 할당하지 않고, 해당 메모리에 처음 쓰기가 발생할 때 페이지를 실제로 할당한다. 이 지연 할당 방식이 **Demand Paging**이다.

덕분에 전체 프로세스가 요청한 메모리 합계가 물리 메모리를 초과해도 시스템이 동작한다. 하지만 실제 접근이 몰리면 메모리가 부족해지고, 그때 OOM Killer가 개입한다.

`/proc/sys/vm/overcommit_memory`로 오버커밋 정책을 설정할 수 있다:

| 값 | 정책 |
|---|---|
| 0 | 휴리스틱 오버커밋 허용 (기본값) |
| 1 | 항상 오버커밋 허용 |
| 2 | 오버커밋 비허용 (엄격하게 가용 메모리 내에서만 할당) |

---

## OOM Killer 동작 과정

**1. 메모리 할당 실패 감지**

페이지 폴트 처리 중 `do_page_alloc()`에서 메모리 할당에 실패하면 커널이 OOM 상황을 판단한다.

**2. OOM 상황 검증**

단순한 메모리 부족인지 진짜 OOM인지 먼저 확인한다. 스왑 공간 여유나 캐시 회수 가능 여부를 먼저 시도한다.

**3. 희생 프로세스 선택**

`oom_killer_select_task()`가 실행되어 각 프로세스의 `oom_score`를 계산하고, 가장 높은 점수의 프로세스를 종료 대상으로 선택한다.

**4. SIGKILL 전송**

선택된 프로세스에 SIGKILL을 보내 강제 종료하고 메모리를 회수한다.

---

## oom_score 계산 방식

`oom_score`는 `/proc/<pid>/oom_score`에서 확인할 수 있다. 점수가 높을수록 OOM Killer에 의해 종료될 가능성이 높다.

점수 계산의 주요 요소:

- **메모리 사용량**: RSS(Resident Set Size) 비율이 클수록 점수 증가 — 가장 큰 비중을 차지
- **실행 시간**: 오래 실행된 프로세스는 점수를 낮춘다 (시스템 데몬일 가능성)
- **root 프로세스**: root 권한 프로세스는 약간 낮은 점수 부여
- **자식 프로세스 메모리**: 자식 프로세스 메모리도 부모 점수에 합산

```bash
# 특정 프로세스 oom_score 확인
cat /proc/<pid>/oom_score

# 전체 프로세스 oom_score 내림차순 확인
for pid in /proc/[0-9]*; do
  score=$(cat "$pid/oom_score" 2>/dev/null)
  comm=$(cat "$pid/comm" 2>/dev/null)
  echo "$score $comm"
done | sort -rn | head -20
```

---

## oom_score_adj로 우선순위 수동 조정

`/proc/<pid>/oom_score_adj`로 특정 프로세스의 OOM 우선순위를 직접 조정할 수 있다.

범위는 -1000 ~ 1000이다:

| 값 | 의미 |
|---|---|
| -1000 | OOM Killer에 의해 절대 종료되지 않음 |
| 0 | 기본값 |
| 1000 | 메모리 부족 시 가장 먼저 종료됨 |

```bash
# nginx가 OOM Killer에 의해 종료되지 않도록 설정
echo -1000 > /proc/$(pidof nginx)/oom_score_adj
```

systemd 서비스에서는 유닛 파일로 영구 설정할 수 있다:

```ini
[Service]
OOMScoreAdjust=-500
```

---

## 커널 로그로 OOM 이벤트 확인

OOM Killer가 실행되면 커널 로그에 상세한 정보가 기록된다.

```bash
# OOM 이벤트 확인
dmesg | grep -i "oom"
dmesg | grep -i "killed process"

# journalctl로 확인
journalctl -k | grep -i oom
```

로그 예시:

```
Out of memory: Kill process 12345 (java) score 872 or sacrifice child
Killed process 12345 (java) total-vm:4096000kB, anon-rss:3907840kB, file-rss:0kB
```

`score 872`가 oom_score이며, `anon-rss`가 실제 사용한 물리 메모리다. 이 정보를 보고 어떤 프로세스가 메모리를 과도하게 점유하고 있었는지 파악한다.

---

## cgroup을 활용한 메모리 격리

현대 리눅스에서는 **cgroup**으로 프로세스 그룹별 메모리 사용량을 제한할 수 있다. OOM 발생 범위를 컨테이너 또는 특정 서비스로 격리하는 핵심 기술이다.

```bash
# cgroup v2에서 메모리 제한 설정
echo "512M" > /sys/fs/cgroup/mygroup/memory.max
```

Docker 컨테이너에서 메모리를 제한하면 내부적으로 cgroup 설정이 적용된다:

```bash
docker run --memory="512m" --memory-swap="1g" myapp
```

Kubernetes에서는 `resources.limits.memory`가 cgroup 메모리 제한으로 변환된다:

```yaml
resources:
  requests:
    memory: "256Mi"
  limits:
    memory: "512Mi"
```

컨테이너가 memory limit을 초과하면 컨테이너 내부 OOM Killer가 동작하거나, 컨테이너 자체가 `OOMKilled` 상태로 종료된다. `kubectl describe pod`로 확인할 수 있다:

```
Last State: Terminated
  Reason: OOMKilled
  Exit Code: 137
```

Exit Code 137은 SIGKILL(128 + 9)을 의미한다.

---

## OOM 방지 전략

**애플리케이션 레벨**

- 메모리 누수 방지: 힙 프로파일러(pprof, JVM heap dump)로 주기적 점검
- 캐시 크기 제한: 무제한 캐시 대신 LRU/TTL 정책 적용
- 커넥션 풀 크기 제한: 데이터베이스 연결이나 HTTP 클라이언트 풀 크기 조정

**시스템 레벨**

```bash
# swappiness 조정 (0: 스왑 최소화, 100: 적극적으로 스왑 사용)
# 메모리 서버에서는 10 정도로 설정하는 경우가 많다
echo 10 > /proc/sys/vm/swappiness

# 영구 설정 (/etc/sysctl.conf)
vm.swappiness = 10
```

**모니터링**

```bash
# 메모리 현황 확인
free -h
cat /proc/meminfo | grep -E "MemTotal|MemFree|MemAvailable|Cached|SwapFree"

# 메모리 사용량 상위 프로세스
ps aux --sort=-%mem | head -10
```

---

## 정리

| 항목 | 내용 |
|---|---|
| OOM 발생 조건 | 물리 메모리 + 스왑 공간 모두 소진 |
| 희생 프로세스 선택 기준 | oom_score 높은 프로세스 (메모리 사용량 기반) |
| 수동 조정 | /proc/pid/oom_score_adj (-1000 ~ 1000) |
| 로그 확인 | dmesg, journalctl -k로 OOM 이벤트 추적 |
| 컨테이너 환경 | cgroup 메모리 제한, OOMKilled(Exit 137) |
| 방지 전략 | 메모리 누수 점검, cgroup 격리, swappiness 조정 |
