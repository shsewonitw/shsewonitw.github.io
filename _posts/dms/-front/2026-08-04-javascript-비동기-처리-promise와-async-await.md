---
layout: post
title: "[Daily morning study] JavaScript 비동기 처리 - Promise와 async/await"
description: >
  #daily morning study
category: 
    - dms
    - dms-frontend
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 왜 비동기 처리가 필요한가

JavaScript는 싱글 스레드 언어다. 한 번에 하나의 작업만 실행할 수 있다는 뜻이다. 만약 네트워크 요청, 파일 읽기, 타이머처럼 시간이 걸리는 작업을 동기적으로 처리하면, 그 동안 브라우저나 서버가 멈춰버린다. 사용자 입장에서는 UI가 굳어버리는 최악의 경험이 된다.

이 문제를 해결하기 위해 JavaScript는 **비동기(asynchronous)** 방식을 사용한다. 오래 걸리는 작업을 백그라운드에 위임하고, 완료되면 결과를 받아 처리하는 구조다.

---

## 콜백(Callback)의 한계

초기에는 비동기 처리를 **콜백 함수**로 해결했다.

```javascript
function fetchUser(id, callback) {
  setTimeout(() => {
    callback({ id, name: 'Alice' });
  }, 1000);
}

fetchUser(1, (user) => {
  console.log(user.name); // Alice
});
```

단순한 경우엔 괜찮지만, 여러 비동기 작업이 순서대로 실행되어야 할 때 문제가 생긴다.

```javascript
fetchUser(1, (user) => {
  fetchPosts(user.id, (posts) => {
    fetchComments(posts[0].id, (comments) => {
      // 이렇게 중첩이 계속된다 → 콜백 지옥(Callback Hell)
    });
  });
});
```

이런 구조는 가독성이 나쁘고, 에러 처리가 복잡하며, 유지보수가 어렵다.

---

## Promise

### 개념

**Promise**는 ES6에서 도입된 비동기 처리를 위한 객체다. "미래에 값이 채워질 상자"라고 생각하면 된다.

Promise는 세 가지 상태를 가진다.

| 상태 | 설명 |
|------|------|
| `pending` | 초기 상태. 아직 이행/거부되지 않음 |
| `fulfilled` | 작업이 성공적으로 완료됨 |
| `rejected` | 작업이 실패함 |

상태는 한 번 변경되면 다시 바뀌지 않는다.

### 기본 사용법

```javascript
const promise = new Promise((resolve, reject) => {
  const success = true;

  if (success) {
    resolve('성공 결과');
  } else {
    reject(new Error('실패 이유'));
  }
});

promise
  .then((result) => {
    console.log(result); // '성공 결과'
  })
  .catch((error) => {
    console.error(error.message);
  })
  .finally(() => {
    console.log('항상 실행됨');
  });
```

### Promise 체이닝

`.then()`은 새로운 Promise를 반환한다. 그래서 체이닝이 가능하다.

```javascript
fetchUser(1)
  .then((user) => fetchPosts(user.id))
  .then((posts) => fetchComments(posts[0].id))
  .then((comments) => {
    console.log(comments);
  })
  .catch((error) => {
    // 어느 단계에서 실패해도 여기서 잡힌다
    console.error(error);
  });
```

콜백 지옥과 비교하면 훨씬 읽기 쉽다.

### Promise 정적 메서드

**Promise.all**

여러 Promise를 동시에 실행하고, 모두 완료될 때 결과를 배열로 받는다. 하나라도 실패하면 즉시 reject된다.

```javascript
const [user, posts] = await Promise.all([
  fetchUser(1),
  fetchPosts(1),
]);
```

**Promise.allSettled**

모두 완료될 때까지 기다리되, 성공/실패 여부를 각각 담아 반환한다. 개별 결과를 확인할 때 유용하다.

```javascript
const results = await Promise.allSettled([
  fetchUser(1),
  fetchPosts(999), // 실패 가능
]);

results.forEach((result) => {
  if (result.status === 'fulfilled') {
    console.log(result.value);
  } else {
    console.error(result.reason);
  }
});
```

**Promise.race**

가장 먼저 완료된 Promise의 결과를 반환한다. 타임아웃 구현에 자주 쓰인다.

```javascript
const timeout = new Promise((_, reject) =>
  setTimeout(() => reject(new Error('Timeout')), 3000)
);

const result = await Promise.race([fetchData(), timeout]);
```

**Promise.any**

하나라도 성공하면 그 결과를 반환한다. 모두 실패할 경우 AggregateError가 발생한다.

---

## async/await

### 개념

**async/await**는 ES2017에서 도입됐다. Promise를 더 읽기 쉬운 동기 코드처럼 작성할 수 있게 해주는 문법적 설탕(syntactic sugar)이다. 내부적으로는 Promise를 그대로 사용한다.

### 기본 사용법

`async` 키워드를 붙인 함수는 항상 Promise를 반환한다. 함수 내부에서 `await`를 쓰면 Promise가 완료될 때까지 해당 줄에서 기다린다.

```javascript
async function loadData() {
  const user = await fetchUser(1);
  const posts = await fetchPosts(user.id);
  const comments = await fetchComments(posts[0].id);
  return comments;
}
```

Promise 체이닝보다 훨씬 직관적으로 읽힌다.

### 에러 처리

`try/catch`를 그대로 사용하면 된다.

```javascript
async function loadData() {
  try {
    const user = await fetchUser(1);
    const posts = await fetchPosts(user.id);
    return posts;
  } catch (error) {
    console.error('에러 발생:', error.message);
    throw error; // 필요하면 다시 던지기
  }
}
```

### 주의: 순차 실행 vs 병렬 실행

`await`를 단순히 나열하면 **순차 실행**이 된다.

```javascript
// 총 2초 소요 (1초 + 1초)
const user = await fetchUser(1);    // 1초 대기
const config = await fetchConfig(); // 1초 대기
```

두 요청이 서로 의존하지 않는다면, `Promise.all`로 **병렬 처리**해야 성능이 좋다.

```javascript
// 총 1초 소요 (동시에 실행)
const [user, config] = await Promise.all([
  fetchUser(1),
  fetchConfig(),
]);
```

이 차이를 모르고 쓰면 의도치 않게 느린 코드가 된다.

---

## 마이크로태스크 큐(Microtask Queue)

Promise의 `.then()`, `async/await`는 **마이크로태스크 큐**를 사용한다. 이벤트 루프에서 마이크로태스크는 매크로태스크(setTimeout, setInterval 등)보다 **먼저** 실행된다.

```javascript
console.log('1');

setTimeout(() => console.log('2'), 0);

Promise.resolve().then(() => console.log('3'));

console.log('4');

// 출력 순서: 1 → 4 → 3 → 2
```

실행 순서를 이해하지 못하면 디버깅이 어려워진다. 순서를 기억해두자.

1. 동기 코드 실행 (콜 스택)
2. 마이크로태스크 큐 비우기 (Promise then, queueMicrotask)
3. 매크로태스크 하나 실행 (setTimeout, setInterval)
4. 다시 마이크로태스크 큐 비우기
5. 반복

---

## 실전 패턴: 재시도(Retry) 로직

네트워크 요청이 실패했을 때 일정 횟수만큼 재시도하는 패턴이다.

```javascript
async function fetchWithRetry(url, retries = 3) {
  for (let i = 0; i < retries; i++) {
    try {
      const response = await fetch(url);
      if (!response.ok) throw new Error(`HTTP ${response.status}`);
      return await response.json();
    } catch (error) {
      if (i === retries - 1) throw error; // 마지막 시도에서 실패하면 예외 전파
      await new Promise((resolve) => setTimeout(resolve, 1000 * (i + 1))); // 지수 백오프
    }
  }
}
```

---

## 정리

| 방식 | 장점 | 단점 |
|------|------|------|
| 콜백 | 간단한 경우 직관적 | 중첩 시 콜백 지옥, 에러 처리 복잡 |
| Promise | 체이닝 가능, 에러 처리 통일 | 디버깅 스택 추적이 다소 불편 |
| async/await | 동기 코드처럼 읽힘, try/catch 사용 가능 | 내부는 Promise이므로 동작 원리 이해 필요 |

현대 JavaScript 개발에서는 거의 대부분 **async/await**를 사용한다. 다만 여러 비동기 작업을 병렬로 실행할 때는 `Promise.all`과 함께 써야 성능을 놓치지 않는다.
