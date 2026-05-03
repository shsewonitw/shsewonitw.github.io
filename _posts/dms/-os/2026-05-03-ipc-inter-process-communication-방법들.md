---
layout: post
title: "[Daily morning study] IPC (Inter-Process Communication) 방법들"
description: >
  #daily morning study
category: 
    - dms
    - dms-os
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## IPC란?

프로세스는 기본적으로 독립된 메모리 공간을 가지기 때문에 서로의 데이터에 직접 접근할 수 없다. IPC(Inter-Process Communication)는 이렇게 격리된 프로세스들이 데이터를 주고받거나 동작을 조율할 수 있도록 OS가 제공하는 메커니즘이다.

IPC가 필요한 상황:
- 여러 프로세스가 협력해서 하나의 작업을 처리할 때
- 생산자-소비자(Producer-Consumer) 구조처럼 데이터를 넘겨야 할 때
- 프로세스 간 이벤트나 시그널을 전달해야 할 때

---

## 주요 IPC 방식

### 1. 파이프 (Pipe)

단방향 데이터 스트림. 부모-자식 프로세스처럼 관련된 프로세스 사이에서 주로 사용한다.

```bash
# 셸에서 쓰는 파이프도 동일한 원리
ls -l | grep ".md"
```

```c
int fd[2];
pipe(fd);  // fd[0]: 읽기 끝, fd[1]: 쓰기 끝

if (fork() == 0) {
    // 자식: 쓰기
    close(fd[0]);
    write(fd[1], "hello", 5);
} else {
    // 부모: 읽기
    close(fd[1]);
    char buf[10];
    read(fd[0], buf, 5);
}
```

특징:
- 단방향 (양방향이 필요하면 파이프 2개 사용)
- 관련 없는 프로세스 간에는 사용 불가 → 이 경우 **Named Pipe(FIFO)** 사용

### 2. Named Pipe (FIFO)

일반 파이프와 달리 파일 시스템에 이름이 있는 파이프. 관계없는 프로세스끼리도 이름으로 찾아서 통신 가능하다.

```bash
mkfifo /tmp/my_pipe

# 프로세스 A: 쓰기
echo "data" > /tmp/my_pipe

# 프로세스 B: 읽기
cat /tmp/my_pipe
```

### 3. 메시지 큐 (Message Queue)

메시지를 큐에 저장하고, 수신자가 꺼내가는 방식. 송신자와 수신자가 동시에 실행 중일 필요가 없다는 점이 파이프와 다르다.

| 항목 | 파이프 | 메시지 큐 |
|------|--------|-----------|
| 방향 | 단방향 | 양방향 가능 |
| 동기화 | 동기 | 비동기 |
| 메시지 단위 | 바이트 스트림 | 타입 있는 메시지 |

```c
// 메시지 큐 생성/접근
int msgid = msgget(key, IPC_CREAT | 0666);

// 송신
msgsnd(msgid, &msg, sizeof(msg.text), 0);

// 수신
msgrcv(msgid, &msg, sizeof(msg.text), 1, 0);
```

### 4. 공유 메모리 (Shared Memory)

여러 프로세스가 동일한 메모리 영역을 직접 읽고 쓰는 방식. **IPC 방법 중 가장 빠르다.**

```
프로세스 A의 가상 주소 공간     프로세스 B의 가상 주소 공간
┌──────────────────┐           ┌──────────────────┐
│      코드        │           │      코드        │
│      스택        │           │      스택        │
│   공유 메모리  ──┼───────────┼── 공유 메모리    │
└──────────────────┘           └──────────────────┘
             ↓                         ↓
             └─────── 물리 메모리 ─────┘
```

속도가 빠른 대신 **동기화 문제**가 발생할 수 있어서 Semaphore나 Mutex와 함께 써야 한다.

```c
// 공유 메모리 생성
int shmid = shmget(key, 1024, IPC_CREAT | 0666);

// 프로세스 주소 공간에 연결
char *data = (char *)shmat(shmid, NULL, 0);

// 읽기/쓰기
sprintf(data, "shared data");

// 연결 해제
shmdt(data);
```

### 5. 소켓 (Socket)

네트워크 통신에 쓰이는 소켓을 같은 머신 내 프로세스 간 통신에도 사용할 수 있다. **Unix Domain Socket**을 쓰면 네트워크 스택을 거치지 않아 빠르다.

```python
# Unix Domain Socket 예시 (Python)
import socket

# 서버
server = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
server.bind("/tmp/my.sock")
server.listen(1)
conn, _ = server.accept()
data = conn.recv(1024)

# 클라이언트
client = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
client.connect("/tmp/my.sock")
client.send(b"hello")
```

TCP/UDP 소켓과 달리 파일 시스템 경로로 연결하기 때문에 같은 호스트 안에서만 동작한다.

### 6. 시그널 (Signal)

데이터를 전달하는 게 아니라 **이벤트 발생을 알리는** 용도. 비동기적으로 전달된다.

```bash
kill -9 1234   # PID 1234 프로세스에 SIGKILL 전송
kill -15 1234  # SIGTERM (정상 종료 요청)
```

```c
#include <signal.h>

void handler(int sig) {
    printf("Received signal: %d\n", sig);
}

signal(SIGINT, handler);  // Ctrl+C 입력 시 handler 실행
```

자주 쓰는 시그널:

| 시그널 | 번호 | 의미 |
|--------|------|------|
| SIGINT | 2 | 인터럽트 (Ctrl+C) |
| SIGKILL | 9 | 강제 종료 (무시 불가) |
| SIGTERM | 15 | 정상 종료 요청 |
| SIGSEGV | 11 | 세그멘테이션 폴트 |
| SIGCHLD | 17 | 자식 프로세스 상태 변화 |

---

## IPC 방식 비교 정리

| 방식 | 속도 | 방향 | 관계없는 프로세스 | 비고 |
|------|------|------|-------------------|------|
| Pipe | 중간 | 단방향 | 불가 | 부모-자식 전용 |
| Named Pipe | 중간 | 단방향 | 가능 | 파일 시스템 경로 |
| Message Queue | 중간 | 양방향 | 가능 | 비동기, 타입 구분 |
| Shared Memory | **빠름** | 양방향 | 가능 | 동기화 별도 필요 |
| Socket | 느림 | 양방향 | 가능 | 네트워크도 가능 |
| Signal | - | 단방향 | 가능 | 데이터 전달 불가 |

---

## 선택 기준

- **속도가 최우선**이고 같은 머신 → 공유 메모리
- **단순한 데이터 흐름**, 부모-자식 관계 → 파이프
- **비동기 메시지 전달** → 메시지 큐
- **분산 환경 확장성** 고려 → 소켓
- **이벤트 알림**만 필요 → 시그널

리눅스 커널 내부에서도 프로세스 간 협력이 필요한 경우 위 메커니즘들을 조합해서 사용한다. 실제 시스템 프로그래밍에서는 공유 메모리 + 세마포어 조합이 성능상 자주 선택되고, 분산 시스템으로 확장을 고려한다면 처음부터 소켓(또는 메시지 브로커) 기반으로 설계하는 편이 낫다.
