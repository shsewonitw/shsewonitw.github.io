---
layout: post
title: "[Daily morning study] I/O 모델 비교 — Blocking, Non-blocking, I/O Multiplexing, Async I/O"
description: >
  #daily morning study
category: 
    - dms
    - dms-os
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## I/O 모델이란

운영체제에서 I/O(Input/Output) 작업은 네트워크, 파일, 소켓 등 외부 자원과의 데이터 교환을 의미한다. I/O 작업은 CPU 연산보다 훨씬 느리기 때문에, 이 대기 시간을 어떻게 처리하느냐에 따라 성능이 크게 달라진다.

Unix/Linux 기준으로 I/O 모델은 크게 4가지로 나뉜다.

---

## 1. Blocking I/O

가장 기본적인 I/O 모델. 프로세스가 I/O 시스템 콜을 호출하면 **데이터가 준비될 때까지 블로킹(대기)** 된다.

```c
int n = read(fd, buf, 1024); // 데이터가 올 때까지 여기서 멈춤
// 이후 코드는 read()가 반환된 후에야 실행됨
```

**흐름**:
1. 프로세스가 `read()` 시스템 콜 호출
2. 커널은 데이터가 준비될 때까지 프로세스를 대기 상태로 전환
3. 데이터가 준비되면 커널이 사용자 공간으로 복사
4. 프로세스가 다시 실행 가능 상태로 전환되며 `read()` 반환

**장점**: 구현이 단순하고 직관적  
**단점**: 하나의 스레드가 하나의 연결만 처리 → 다중 연결을 위해 멀티스레드 필요 → 메모리/컨텍스트 스위칭 오버헤드 증가

---

## 2. Non-blocking I/O

시스템 콜을 호출했을 때 데이터가 준비되지 않았다면 **즉시 `EAGAIN` 또는 `EWOULDBLOCK` 에러를 반환**한다.

```c
fcntl(fd, F_SETFL, O_NONBLOCK); // 소켓을 논블로킹으로 설정

while (true) {
    int n = read(fd, buf, 1024);
    if (n == -1 && errno == EAGAIN) {
        // 데이터 아직 없음 → 다른 작업 또는 재시도
        continue;
    }
    // 데이터 처리
    break;
}
```

**흐름**:
1. 프로세스가 `read()` 호출
2. 데이터가 없으면 즉시 `-1` 반환 (에러: EAGAIN)
3. 프로세스는 반복적으로 재시도 (폴링, Polling)
4. 데이터가 준비되면 정상 반환

**장점**: 블로킹 없이 다른 작업 수행 가능  
**단점**: 데이터가 준비되었는지 계속 확인하는 **바쁜 대기(Busy Waiting)** → CPU 낭비

---

## 3. I/O Multiplexing

`select()`, `poll()`, `epoll()` 같은 시스템 콜을 사용해 **여러 파일 디스크립터를 동시에 감시**한다. 하나의 스레드로 수천 개의 소켓 이벤트를 처리할 수 있다.

```c
// epoll 예시
int epfd = epoll_create1(0);

struct epoll_event ev;
ev.events = EPOLLIN; // 읽기 가능 이벤트 감시
ev.data.fd = sockfd;
epoll_ctl(epfd, EPOLL_CTL_ADD, sockfd, &ev);

struct epoll_event events[MAX_EVENTS];
while (true) {
    int n = epoll_wait(epfd, events, MAX_EVENTS, -1); // 이벤트 발생 대기
    for (int i = 0; i < n; i++) {
        read(events[i].data.fd, buf, sizeof(buf));
    }
}
```

### select, poll, epoll 비교

| 항목 | select | poll | epoll |
|------|--------|------|-------|
| 최대 FD 수 | 1024 (FD_SETSIZE) | 제한 없음 | 제한 없음 |
| 시간 복잡도 | O(n) | O(n) | O(1) |
| FD 전달 방식 | 매 호출마다 전체 FD 복사 | 매 호출마다 전체 FD 복사 | 관심 FD 등록 후 이벤트만 통보 |
| 지원 환경 | 모든 UNIX | POSIX 표준 | Linux 2.6+ |

epoll이 select/poll보다 효율적인 이유는 **이벤트 기반(Event-driven)** 방식이기 때문이다. select/poll은 매번 모든 FD를 순회하지만, epoll은 변화가 있는 FD만 반환한다.

**장점**: 하나의 스레드로 다중 연결 처리 가능, 효율적  
**단점**: 여전히 데이터 복사(커널 → 사용자 공간)는 블로킹으로 발생

---

## 4. Async I/O (비동기 I/O)

`aio_read()` (POSIX AIO) 또는 Linux의 `io_uring` 같은 API를 사용한다. I/O 요청을 **커널에 맡기고 즉시 반환**한 뒤, I/O 완료 시 **콜백 또는 시그널로 통보**받는다.

```c
// POSIX AIO 예시
struct aiocb cb = {0};
cb.aio_fildes = fd;
cb.aio_buf    = buf;
cb.aio_nbytes = 1024;
cb.aio_offset = 0;

aio_read(&cb); // 즉시 반환, 커널이 백그라운드에서 I/O 수행

// 다른 작업 수행 ...

// 완료 확인
while (aio_error(&cb) == EINPROGRESS)
    ;

ssize_t n = aio_return(&cb); // 결과 수집
```

**흐름**:
1. 프로세스가 `aio_read()` 호출 → 즉시 반환
2. 커널이 백그라운드에서 I/O 수행
3. I/O 완료 시 시그널(SIGIO) 또는 콜백으로 알림
4. 프로세스는 `aio_return()`으로 결과 수집

**장점**: 완전한 비동기 처리, CPU 낭비 없음  
**단점**: API가 복잡하고 지원 범위 제한적 (소켓에서 불완전한 경우 있음)

---

## 전체 비교 요약

| 모델 | 1단계 대기 | 폴링 필요 | 다중 FD | 커널 복잡도 |
|------|-----------|-----------|---------|------------|
| Blocking I/O | 블로킹 | 불필요 | X | 낮음 |
| Non-blocking I/O | 비블로킹 | 필요 (Busy Wait) | X | 낮음 |
| I/O Multiplexing | 이벤트 대기 | 불필요 | O | 중간 |
| Async I/O | 즉시 반환 | 불필요 | O | 높음 |

---

## 실제 활용 사례

- **Blocking I/O**: 단순 스크립트, 동기 파일 처리
- **Non-blocking I/O**: 단독 소켓 상태 확인
- **I/O Multiplexing (epoll)**: Nginx, Redis, Node.js 이벤트 루프, 고성능 서버
- **Async I/O (io_uring)**: 최신 고성능 Linux 서버, 데이터베이스 스토리지 엔진

특히 Node.js의 이벤트 루프는 libuv를 통해 epoll(Linux), kqueue(macOS), IOCP(Windows)를 추상화한 I/O Multiplexing 위에서 동작한다.

---

## Blocking / Async 관점에서 다시 보기

Unix I/O는 두 단계로 나뉜다.

1. **데이터 준비 단계**: 커널이 외부 데이터(네트워크 패킷 등)를 커널 버퍼에 저장
2. **데이터 복사 단계**: 커널 버퍼 → 사용자 프로세스 버퍼로 복사

| 모델 | 1단계 (준비) | 2단계 (복사) |
|------|------------|------------|
| Blocking I/O | 블로킹 | 블로킹 |
| Non-blocking I/O | 비블로킹 (EAGAIN 반환) | 블로킹 |
| I/O Multiplexing | 이벤트 대기 후 블로킹 | 블로킹 |
| Async I/O | 비블로킹 | 커널이 처리 후 알림 |

Non-blocking I/O와 I/O Multiplexing은 흔히 "비동기"라고 부르지만, 정확히는 **비블로킹 동기(Synchronous Non-blocking) I/O**다. 데이터 복사 단계에서는 여전히 프로세스가 직접 관여하기 때문이다. Async I/O만이 진정한 의미의 비동기로, 복사까지 커널이 처리하고 완료 후 알려준다.
