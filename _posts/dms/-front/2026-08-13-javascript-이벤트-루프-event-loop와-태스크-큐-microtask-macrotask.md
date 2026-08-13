---
layout: post
title: "[Daily morning study] JavaScript 이벤트 루프(Event Loop)와 태스크 큐(Microtask vs Macrotask)"
description: >
  #daily morning study
category: 
    - dms
    - dms-frontend
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 브라우저에서 JavaScript가 실행되는 방식

JavaScript는 싱글 스레드 언어다. 동시에 하나의 작업만 실행할 수 있다. 그런데 어떻게 비동기 작업(타이머, 네트워크 요청, 이벤트 핸들러)이 멈추지 않고 동작하는 걸까? 그 핵심이 바로 **이벤트 루프(Event Loop)** 다.

브라우저 환경에서 JavaScript 실행을 구성하는 요소들은 다음과 같다.

| 구성 요소 | 설명 |
|-----------|------|
| **Call Stack** | 현재 실행 중인 함수들이 쌓이는 스택 |
| **Web APIs** | 브라우저가 제공하는 비동기 API (setTimeout, fetch, DOM 이벤트 등) |
| **Microtask Queue** | Promise `.then()`, `queueMicrotask()`, `MutationObserver` 콜백이 쌓이는 큐 |
| **Macrotask Queue** | `setTimeout`, `setInterval`, `setImmediate`, I/O 콜백이 쌓이는 큐 |
| **Event Loop** | Call Stack이 비면 큐에서 콜백을 꺼내 실행하는 루프 |

---

## 이벤트 루프 동작 순서

이벤트 루프의 핵심 규칙은 하나다.

> **Call Stack이 비면, Microtask Queue를 모두 처리한 뒤, Macrotask Queue에서 하나를 꺼낸다.**

구체적인 순서는 아래와 같다.

1. Call Stack에서 현재 실행 중인 코드를 모두 처리한다
2. **Microtask Queue가 빌 때까지** 모든 마이크로태스크를 처리한다
3. 필요하면 렌더링(화면 업데이트)을 수행한다
4. Macrotask Queue에서 **하나**를 꺼내 실행한다
5. 다시 1번으로 돌아간다

이 반복이 "이벤트 루프" 다.

---

## Microtask vs Macrotask 비교

### Microtask (마이크로태스크)

- Promise의 `.then()`, `.catch()`, `.finally()` 콜백
- `queueMicrotask()` 로 직접 등록한 콜백
- `MutationObserver` 콜백
- `async/await` 내부의 `await` 이후 코드

**특징**: 현재 태스크가 끝나면 즉시, 다음 매크로태스크 전에 전부 처리된다.

### Macrotask (매크로태스크)

- `setTimeout(() => {}, 0)`
- `setInterval()`
- `requestAnimationFrame()` (엄밀히는 별도 큐지만 렌더링 후에 실행)
- DOM 이벤트 핸들러 (click, keydown 등)
- `XMLHttpRequest` 콜백

**특징**: 이벤트 루프가 한 번 돌 때마다 하나씩만 처리된다.

---

## 실행 순서 예제

```javascript
console.log('1');

setTimeout(() => console.log('2'), 0); // Macrotask

Promise.resolve()
  .then(() => console.log('3'))        // Microtask
  .then(() => console.log('4'));       // Microtask

console.log('5');
```

**출력 순서**: `1` → `5` → `3` → `4` → `2`

**분석**:
1. `console.log('1')` — 동기, Call Stack에서 즉시 실행
2. `setTimeout(...)` — Web API로 위임, 0ms 후 Macrotask Queue에 추가
3. `Promise.resolve().then(...)` — Microtask Queue에 콜백 두 개 추가
4. `console.log('5')` — 동기, Call Stack에서 즉시 실행
5. Call Stack이 비자 Microtask Queue 처리: `3`, `4` 출력
6. Macrotask Queue에서 하나 꺼내 실행: `2` 출력

---

## Microtask 내에서 새로운 Microtask 추가

마이크로태스크 내부에서 또 다른 마이크로태스크를 추가하면, 그 마이크로태스크도 현재 사이클에서 처리된다.

```javascript
Promise.resolve()
  .then(() => {
    console.log('microtask 1');
    Promise.resolve().then(() => console.log('microtask 2')); // 새로운 마이크로태스크
  })
  .then(() => console.log('microtask 3'));

setTimeout(() => console.log('macrotask'), 0);
```

**출력 순서**: `microtask 1` → `microtask 2` → `microtask 3` → `macrotask`

`microtask 2`가 Macrotask 전에 처리되는 이유는, Microtask Queue가 완전히 빌 때까지 이벤트 루프가 Macrotask로 넘어가지 않기 때문이다.

---

## async/await와 이벤트 루프

`async/await`는 내부적으로 Promise와 Microtask를 사용한다. `await` 키워드는 나머지 함수 실행을 Microtask Queue에 예약하고 Call Stack을 반환한다.

```javascript
async function run() {
  console.log('A');
  await Promise.resolve();
  console.log('B'); // Microtask Queue에 예약됨
}

run();
console.log('C');
```

**출력 순서**: `A` → `C` → `B`

`await` 이후 코드(`B`)는 마이크로태스크로 처리되기 때문에, `run()` 호출 이후의 동기 코드(`C`)가 먼저 실행된다.

---

## 렌더링 타이밍

브라우저는 Microtask를 모두 처리한 뒤, 다음 Macrotask 전에 필요하면 화면을 다시 그린다 (렌더링). 때문에 마이크로태스크를 과도하게 쌓으면 렌더링이 지연될 수 있다.

```javascript
// 나쁜 예: 마이크로태스크가 너무 많아 렌더링 차단
function infiniteMicrotask() {
  Promise.resolve().then(infiniteMicrotask);
}
infiniteMicrotask(); // 브라우저가 멈춤
```

반면 `setTimeout(..., 0)`을 사용하면 매크로태스크로 처리되기 때문에 렌더링 사이에 실행된다.

---

## 정리: 우선순위

```
동기 코드 (Call Stack)
    ↓
Microtask Queue (전부 소진될 때까지)
    ↓
렌더링 (필요한 경우)
    ↓
Macrotask Queue (하나씩)
    ↓
다시 반복
```

실무에서 이 순서가 중요한 이유는 DOM 업데이트 타이밍, 애니메이션 프레임 제어, 성능 최적화에 직접 영향을 주기 때문이다. `setTimeout(..., 0)`으로 다음 사이클에 실행을 미루거나, `queueMicrotask()`로 렌더링 전에 처리할 작업을 정밀하게 스케줄링하는 패턴이 여기서 나온다.
