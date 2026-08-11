---
layout: post
title: "[Daily morning study] Linux namespaces와 cgroups — 컨테이너 격리 기술의 기반"
description: >
  #daily morning study
category: 
    - dms
    - dms-os
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 컨테이너는 어떻게 격리되는가

Docker 같은 컨테이너 기술이 "가상 머신처럼 격리되지만 훨씬 가볍다"고 말하는 이유는 VM처럼 별도의 커널을 띄우지 않기 때문이다. 컨테이너는 **호스트 OS의 커널을 공유**하면서, 리눅스 커널이 제공하는 두 가지 핵심 기능으로 격리와 제한을 구현한다.

- **namespaces** — 프로세스가 보는 시스템 뷰(view)를 격리
- **cgroups (control groups)** — 자원(CPU, 메모리 등) 사용량을 제한

---

## namespaces — 무엇이 보이는지를 격리한다

namespace는 커널 자원을 분리된 집합으로 포장한다. 같은 namespace 안의 프로세스끼리는 서로 보이고, 다른 namespace의 프로세스는 보이지 않는다.

### 주요 namespace 종류

| namespace | 격리 대상 | 예시 |
|-----------|-----------|------|
| `pid` | 프로세스 ID | 컨테이너 안의 init 프로세스가 PID 1로 보임 |
| `net` | 네트워크 인터페이스, 라우팅 테이블, 포트 | 컨테이너마다 독립된 eth0, 독립된 127.0.0.1 |
| `mnt` | 마운트 포인트(파일시스템 뷰) | 컨테이너가 `/` 로 보는 경로가 호스트와 다름 |
| `uts` | hostname, domainname | 컨테이너별로 다른 hostname 설정 가능 |
| `ipc` | System V IPC, POSIX 메시지 큐 | 컨테이너 간 공유 메모리 격리 |
| `user` | UID/GID 매핑 | 컨테이너 안 root가 호스트 비루트 UID에 매핑 |
| `cgroup` | cgroup 루트 뷰 | 컨테이너가 자신의 cgroup 계층을 독립적으로 봄 |

### 직접 확인해보기

```bash
# 현재 프로세스의 namespace 확인
ls -la /proc/self/ns/

# unshare로 새 PID namespace에서 쉘 실행
sudo unshare --pid --mount-proc --fork bash
# 이 쉘 안에서 ps aux하면 프로세스가 극소수만 보인다
```

`/proc/<pid>/ns/` 디렉토리에는 해당 프로세스가 속한 각 namespace에 대한 심볼릭 링크가 있다. 두 프로세스가 같은 namespace에 속하면 같은 inode 번호를 가리킨다.

---

## pid namespace 상세

새 pid namespace를 만들면 그 안의 첫 번째 프로세스는 PID 1이 된다. 이 프로세스가 죽으면 namespace 안의 모든 프로세스에게 SIGKILL이 전파된다. Docker 컨테이너가 `CMD`로 지정한 프로세스가 PID 1이 되는 이유가 여기 있다.

```
호스트 PID 뷰        컨테이너 내부 PID 뷰
-----------         -------------------
1 (systemd)         1 (nginx)
...
8423 (containerd)
8450 (nginx)   <--> 1 (nginx)
8451 (nginx)   <--> 2 (nginx worker)
```

컨테이너 안에서는 nginx가 PID 1로 보이지만, 호스트에서는 PID 8450으로 보인다.

---

## net namespace 상세

각 net namespace는 자체 네트워크 스택을 가진다.

- 독립된 네트워크 인터페이스 (veth pair로 호스트와 연결)
- 독립된 라우팅 테이블
- 독립된 iptables 규칙
- 독립된 소켓

컨테이너가 `80번 포트`를 쓸 수 있는 이유가 여기 있다. 컨테이너 네트워크 namespace 안의 80 포트는 호스트의 80 포트와 다른 포트다. 호스트 포트와 연결하려면 포트 포워딩(`-p 8080:80`)이 필요하다.

```
[컨테이너 net ns]         [호스트 net ns]
  eth0 (172.17.0.2)  <--- veth pair ---> docker0 브리지 (172.17.0.1)
  :80                                    :8080 (NAT via iptables)
```

---

## cgroups — 얼마나 쓸 수 있는지를 제한한다

namespace가 "무엇이 보이는가"를 제어한다면, cgroups는 "얼마나 쓸 수 있는가"를 제어한다. 프로세스 그룹에 자원 사용 한도를 설정하는 커널 기능이다.

### 주요 cgroup 서브시스템

| 서브시스템 | 제어 대상 |
|-----------|-----------|
| `cpu` | CPU 사용 시간 (shares, quota/period) |
| `cpuset` | 특정 CPU 코어, NUMA 노드 할당 |
| `memory` | 메모리 사용 한도, OOM 제어 |
| `blkio` | 블록 I/O 대역폭, IOPS |
| `net_cls` | 네트워크 패킷에 태그를 붙여 tc(traffic control) 적용 |
| `pids` | 프로세스 수 제한 |

### cgroup v1 vs cgroup v2

현재 cgroup에는 두 버전이 있다.

- **cgroup v1**: 서브시스템마다 독립된 계층. `/sys/fs/cgroup/cpu/`, `/sys/fs/cgroup/memory/` 등에 별도로 마운트된다. 설정이 분산되어 일관성 유지가 어렵다.
- **cgroup v2**: 단일 통합 계층. `/sys/fs/cgroup/` 하나에 모든 서브시스템이 모인다. 최신 리눅스 배포판과 컨테이너 런타임은 v2 중심으로 이동 중.

```bash
# cgroup v2 여부 확인
mount | grep cgroup2
# 또는
stat -f /sys/fs/cgroup | grep "Type"
```

### cgroup 직접 조작

```bash
# 메모리 제한 그룹 생성 및 설정
mkdir /sys/fs/cgroup/memory/mygroup
echo 104857600 > /sys/fs/cgroup/memory/mygroup/memory.limit_in_bytes  # 100MB

# 현재 쉘 프로세스를 이 그룹에 추가
echo $$ > /sys/fs/cgroup/memory/mygroup/cgroup.procs
```

이후 이 쉘에서 실행한 프로세스는 100MB 메모리 한도 아래에서 동작하며, 초과 시 OOM killer가 개입한다.

---

## Docker가 이를 사용하는 방식

`docker run --cpus="0.5" --memory="256m" nginx` 실행 시 내부적으로 일어나는 일:

1. **namespace 생성**: pid, net, mnt, uts, ipc namespace를 새로 생성
2. **cgroup 생성**: `/sys/fs/cgroup/.../docker/<container_id>/` 경로에 cgroup 생성, CPU와 메모리 한도 설정
3. **루트 파일시스템 마운트**: 컨테이너 이미지 레이어를 overlay 파일시스템으로 mnt namespace에 마운트
4. **veth pair 생성**: 컨테이너 net namespace에 eth0, 호스트에 veth를 연결
5. **프로세스 실행**: 준비된 namespace와 cgroup 안에서 nginx 실행

---

## 컨테이너 vs 가상 머신 격리 비교

| 구분 | VM | 컨테이너 |
|------|----|---------|
| 커널 | 게스트 OS 자체 커널 | 호스트 커널 공유 |
| 격리 수준 | 하드웨어 수준 (hypervisor) | OS 수준 (namespace/cgroups) |
| 부팅 시간 | 수십 초 | 수백 밀리초 |
| 오버헤드 | 높음 (메모리, CPU 모두) | 낮음 |
| 보안 경계 | 강함 | namespace 탈출 취약점 가능 |

컨테이너는 VM보다 가볍지만 커널을 공유하는 만큼 커널 취약점이 호스트까지 영향을 줄 수 있다. 이를 보완하기 위해 seccomp, AppArmor/SELinux, 읽기 전용 루트 파일시스템 등을 추가로 적용한다.

---

## seccomp — 시스템 콜 필터링

namespace와 cgroups 외에 컨테이너 보안을 강화하는 또 다른 커널 기능이다. **seccomp(Secure Computing Mode)**는 프로세스가 호출할 수 있는 시스템 콜을 제한한다.

```bash
# Docker 기본 seccomp 프로필 적용 확인
docker inspect <container_id> | grep -i seccomp
```

Docker는 기본적으로 300개 이상의 시스템 콜 중 위험한 약 40개를 차단하는 기본 seccomp 프로필을 적용한다. `--privileged` 플래그를 주면 이 모든 제한이 해제된다.

---

## 정리

- **namespaces**: 프로세스가 보는 시스템 자원 뷰를 분리. pid, net, mnt, uts, ipc, user, cgroup 7가지.
- **cgroups**: 프로세스 그룹의 CPU, 메모리, I/O 등 자원 사용량에 상한선 설정.
- 두 기능의 조합으로 컨테이너는 별도 커널 없이 격리 환경을 구현한다.
- VM 대비 가볍지만 커널 공유로 인한 보안 경계가 약하므로, seccomp, AppArmor 등 추가 보호를 함께 사용한다.
