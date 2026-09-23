---
layout: post
title: "[Daily morning study] Backpressure(백프레셔)와 리액티브 스트림 개념"
description: >
  #daily morning study
category: 
    - dms
    - dms-backend
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## Backpressure란?

생산자(Producer)가 소비자(Consumer)보다 데이터를 훨씬 빠르게 생성할 때 발생하는 문제를 **백프레셔**라고 한다. 소비자 쪽에서 처리 속도가 따라가지 못하면 버퍼가 가득 차고, 결국 메모리 초과나 데이터 유실이 발생한다.

예시:

- 초당 100만 건의 로그를 생성하는 서버 → 초당 1만 건만 처리 가능한 소비자
- 빠른 데이터베이스 쿼리 결과 → 느린 HTTP 응답 처리
- IoT 센서 데이터 스트림 → 배치 처리 파이프라인

```
Producer ──────────────────────▶ Consumer
  100 msg/sec                     10 msg/sec
                    ↓
              Buffer overflow → OOM or data loss
```

---

## Reactive Streams 표준

JVM 생태계에서 백프레셔를 표준화하기 위해 **Reactive Streams** 스펙이 만들어졌다 (2013년, Netflix/Pivotal/Typesafe 주도). Java 9부터는 `java.util.concurrent.Flow`로 표준 라이브러리에 포함됐다.

### 핵심 인터페이스 4가지

| 인터페이스 | 역할 |
| --- | --- |
| `Publisher<T>` | 데이터를 생성하고 방출하는 소스 |
| `Subscriber<T>` | 데이터를 구독하고 소비하는 쪽 |
| `Subscription` | Publisher와 Subscriber 사이의 계약 (요청량 제어) |
| `Processor<T,R>` | Publisher이자 Subscriber (중간 변환 단계) |

### 동작 흐름

```
1. Subscriber → Publisher.subscribe(subscriber)
2. Publisher → Subscriber.onSubscribe(subscription)
3. Subscriber → Subscription.request(n)  ← 핵심: n개만 달라
4. Publisher → Subscriber.onNext(item)  (n번 반복)
5. 완료 시 → Subscriber.onComplete()
6. 오류 시 → Subscriber.onError(throwable)
```

`request(n)`이 백프레셔의 핵심이다. 소비자가 처리 가능한 개수만큼만 요청하기 때문에 생산자가 과도하게 데이터를 밀어넣지 못한다.

---

## 백프레셔 전략

`request(n)` 방식이 이상적이지만, 스트림 특성상 소비자가 요청량을 정확히 제어하기 어려운 경우도 있다. 이때 사용하는 전략들:

### 1. Drop (버리기)

버퍼가 가득 차면 새로 들어오는 데이터를 그냥 버린다.

- 장점: 메모리 보호
- 단점: 데이터 유실
- 적합: 실시간 센서 데이터, 최신값이 중요한 경우

### 2. Buffer (버퍼링)

처리될 때까지 대기열에 쌓는다. 버퍼 크기를 설정해야 한다.

- 장점: 데이터 유실 없음
- 단점: 메모리 사용 증가, 지연 발생
- 적합: 순서 보장이 중요한 금융 이벤트

### 3. Latest (최신값만 유지)

소비자가 바쁠 때 중간 값들을 버리고 가장 최신값만 유지한다.

- 장점: 항상 최신 상태 반영
- 단점: 중간 값 유실
- 적합: UI 업데이트, 주식 호가 표시

### 4. Error / Fail

버퍼 초과 시 예외를 던져 파이프라인 자체를 실패로 처리한다.

- 장점: 명시적인 문제 인지
- 단점: 전체 스트림 중단
- 적합: 엄격한 데이터 처리 요구사항

### 5. Block (블로킹)

소비자가 준비될 때까지 생산자 쪽을 블로킹한다. Reactive 철학에 반하지만 특수한 경우 사용.

---

## Project Reactor 예시 (Spring WebFlux)

Spring WebFlux는 Project Reactor를 기반으로 동작한다. `Flux`(0~N개)와 `Mono`(0~1개)가 핵심 타입이다.

### 기본 백프레셔 시연

```java
Flux.range(1, 1_000_000)
    .log()
    .subscribe(new BaseSubscriber<Integer>() {
        @Override
        protected void hookOnSubscribe(Subscription subscription) {
            // 처음에 10개만 요청
            request(10);
        }

        @Override
        protected void hookOnNext(Integer value) {
            System.out.println("Received: " + value);
            // 처리 후 다음 10개 요청
            if (value % 10 == 0) {
                request(10);
            }
        }
    });
```

### onBackpressureDrop 전략 적용

```java
Flux.interval(Duration.ofMillis(1))   // 1ms마다 데이터 생성
    .onBackpressureDrop(dropped ->
        log.warn("Dropped: {}", dropped))
    .publishOn(Schedulers.single())
    .doOnNext(i -> {
        // 느린 처리 시뮬레이션
        Thread.sleep(10);
        process(i);
    })
    .subscribe();
```

### onBackpressureBuffer with limited size

```java
Flux.interval(Duration.ofMillis(1))
    .onBackpressureBuffer(
        100,                        // 최대 버퍼 크기
        dropped -> log.warn("Buffer full, dropped: {}", dropped),
        BufferOverflowStrategy.DROP_OLDEST  // 오래된 것부터 제거
    )
    .publishOn(Schedulers.single())
    .subscribe(this::process);
```

---

## Cold vs Hot Publisher

백프레셔를 이해할 때 Cold/Hot 개념도 중요하다.

| 구분 | Cold Publisher | Hot Publisher |
| --- | --- | --- |
| 데이터 생성 시점 | 구독 시작 시 | 구독과 무관하게 계속 생성 |
| 예시 | HTTP 요청, 파일 읽기 | 주식 시세, IoT 센서, 이벤트 버스 |
| 백프레셔 적용 | 자연스럽게 적용 | 명시적인 전략 필요 |
| 지각 구독자 | 처음부터 데이터 받음 | 구독 이후 데이터만 받음 |

Hot Publisher는 소비자 유무와 관계없이 데이터를 방출하기 때문에 백프레셔 처리가 더 까다롭다.

---

## Spring WebFlux에서의 실제 흐름

```
HTTP Request
    ↓
RouterFunction / Controller
    ↓
Flux<ResponseBody>  ← 여기서 백프레셔 적용
    ↓
Netty (Non-blocking I/O)
    ↓
TCP Buffer ← 클라이언트가 느리면 여기서 자연스럽게 백프레셔 전파
    ↓
HTTP Client
```

WebFlux에서는 TCP 소켓의 수신 버퍼가 가득 차면 Netty가 읽기를 멈추고, 그 신호가 Reactor 스트림 전체에 역방향으로 전파된다. 이것이 end-to-end 백프레셔다.

---

## RxJava와의 비교

| 항목 | Project Reactor | RxJava 3 |
| --- | --- | --- |
| 핵심 타입 | `Flux`, `Mono` | `Flowable`, `Observable`, `Single` |
| 백프레셔 지원 | 항상 지원 | `Flowable`만 지원, `Observable`은 미지원 |
| Java 9 Flow 호환 | 지원 | 지원 |
| 주 사용처 | Spring WebFlux | Android, 일반 JVM |

RxJava에서 `Observable`은 백프레셔를 지원하지 않아 `MissingBackpressureException`이 발생할 수 있다. 백프레셔가 필요하면 반드시 `Flowable`을 사용해야 한다.

---

## 정리

- 백프레셔는 생산자-소비자 속도 불균형을 제어하는 메커니즘
- Reactive Streams 표준이 `request(n)`으로 이를 명시적으로 제어
- 버퍼링, 드롭, 최신값 유지 등의 전략 중 도메인 요구사항에 맞는 것 선택
- Spring WebFlux + Project Reactor는 TCP 레벨까지 end-to-end 백프레셔를 지원
- Hot Publisher 환경에서는 반드시 명시적인 백프레셔 전략을 설정해야 데이터 유실이나 OOM을 막을 수 있다
