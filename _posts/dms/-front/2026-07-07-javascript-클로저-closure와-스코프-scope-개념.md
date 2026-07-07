---
layout: post
title: "[Daily morning study] JavaScript 클로저(Closure)와 스코프(Scope) 개념"
description: >
  #daily morning study
category: 
    - dms
    - dms-frontend
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 스코프(Scope)란

스코프는 변수나 함수가 **유효하게 참조될 수 있는 범위**다. JavaScript에서 스코프는 코드가 실행되는 시점이 아닌, **코드가 작성된 위치**에 따라 정적으로 결정된다. 이를 **렉시컬 스코프(Lexical Scope)** 또는 정적 스코프라고 한다.

### 스코프의 종류

| 스코프 | 범위 | 생성 시점 |
|--------|------|-----------|
| 전역 스코프 | 어디서든 접근 가능 | 스크립트 최상위 |
| 함수 스코프 | 해당 함수 내부 | 함수 호출 시 |
| 블록 스코프 | `{}` 블록 내부 | `let`, `const` 사용 시 |

```js
var x = 'global';

function outer() {
  var y = 'outer';

  function inner() {
    var z = 'inner';
    console.log(x); // 'global' — 상위 스코프 참조 가능
    console.log(y); // 'outer'  — 상위 스코프 참조 가능
    console.log(z); // 'inner'
  }

  inner();
  // console.log(z); // ReferenceError — 하위 스코프는 참조 불가
}
```

### var vs let/const 스코프 차이

`var`는 함수 스코프를 따르고, `let`/`const`는 블록 스코프를 따른다.

```js
function testScope() {
  if (true) {
    var a = 'var';   // 함수 전체에서 접근 가능
    let b = 'let';   // 블록 내부에서만 접근 가능
  }
  console.log(a); // 'var'
  console.log(b); // ReferenceError
}
```

---

## 스코프 체인(Scope Chain)

함수가 중첩될 때, 내부 함수는 자신의 스코프부터 시작해 외부 스코프를 순서대로 탐색한다. 이 연결 구조를 **스코프 체인**이라 한다.

```
전역 스코프
  └── outer 스코프
        └── inner 스코프
```

변수를 찾을 때 현재 스코프에 없으면 상위 스코프를 타고 올라가며 탐색하고, 전역까지 없으면 `ReferenceError`가 발생한다.

---

## 클로저(Closure)란

클로저는 **함수가 자신이 선언된 렉시컬 스코프를 기억하고, 그 스코프 밖에서 호출되어도 해당 스코프에 접근할 수 있는 함수**다.

더 간단히 말하면: **함수 + 함수가 선언된 당시의 외부 변수 환경**의 조합이 클로저다.

```js
function makeCounter() {
  let count = 0; // makeCounter 스코프의 변수

  return function () {
    count++;
    return count;
  };
}

const counter = makeCounter();
console.log(counter()); // 1
console.log(counter()); // 2
console.log(counter()); // 3
```

`makeCounter()`가 종료된 후에도 반환된 함수는 `count` 변수를 계속 참조할 수 있다. 이게 클로저다. `count`는 외부에서 직접 접근할 수 없고, 반환된 함수를 통해서만 조작된다.

---

## 클로저가 유용한 이유

### 1. 데이터 은닉 (캡슐화)

외부에서 직접 접근하지 못하도록 변수를 숨길 수 있다.

```js
function createUser(name) {
  let _name = name; // private처럼 동작

  return {
    getName: () => _name,
    setName: (newName) => { _name = newName; },
  };
}

const user = createUser('Alice');
console.log(user.getName()); // 'Alice'
user.setName('Bob');
console.log(user.getName()); // 'Bob'
// user._name 은 undefined
```

### 2. 함수 팩토리

공통 로직을 공유하면서 설정값만 다른 함수를 여러 개 만들 때 유용하다.

```js
function makeMultiplier(multiplier) {
  return (num) => num * multiplier;
}

const double = makeMultiplier(2);
const triple = makeMultiplier(3);

console.log(double(5)); // 10
console.log(triple(5)); // 15
```

### 3. 부분 적용(Partial Application)

인수를 미리 일부 채워놓은 함수를 만들 수 있다.

```js
function add(a, b) {
  return a + b;
}

function partial(fn, ...presetArgs) {
  return (...laterArgs) => fn(...presetArgs, ...laterArgs);
}

const add5 = partial(add, 5);
console.log(add5(3)); // 8
console.log(add5(10)); // 15
```

---

## 자주 나오는 클로저 함정

### 반복문과 var 조합

```js
// 의도: 0, 1, 2를 1초 간격으로 출력
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 1000 * i);
}
// 실제 출력: 3, 3, 3
```

`var i`는 함수 스코프이므로 모든 콜백이 같은 `i`를 참조한다. setTimeout이 실행될 때 이미 루프가 끝나 `i === 3`이 됐다.

**해결 방법 1: `let` 사용**

```js
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 1000 * i);
}
// 출력: 0, 1, 2
```

`let`은 블록 스코프이므로 반복마다 새로운 `i`가 생성된다.

**해결 방법 2: IIFE로 값 캡처**

```js
for (var i = 0; i < 3; i++) {
  (function (j) {
    setTimeout(() => console.log(j), 1000 * j);
  })(i);
}
// 출력: 0, 1, 2
```

---

## 메모리 누수 주의

클로저는 외부 변수를 계속 참조하기 때문에 **불필요하게 오래 살아있으면 메모리 누수**가 발생할 수 있다.

```js
function attachHandler() {
  const largeData = new Array(1000000).fill('data');

  document.getElementById('btn').addEventListener('click', () => {
    // largeData를 실제로 사용하지 않지만 클로저로 참조 유지
    console.log('clicked');
  });
}
```

이벤트 리스너를 제거하지 않으면 `largeData`가 GC되지 않는다. 사용이 끝나면 `removeEventListener`로 리스너를 정리해야 한다.

---

## 실행 컨텍스트와의 관계

클로저를 깊이 이해하려면 **실행 컨텍스트(Execution Context)** 와 **환경 레코드(Environment Record)** 개념이 필요하다.

함수가 생성될 때, 함수 객체 내부에 `[[Environment]]`라는 내부 슬롯이 생긴다. 여기에 함수가 선언된 당시의 렉시컬 환경(외부 변수들의 참조)이 저장된다. 함수가 나중에 호출될 때 이 `[[Environment]]`를 통해 외부 변수에 접근하는 것이 클로저의 동작 원리다.

```
함수 객체
  └── [[Environment]] → 선언 당시 렉시컬 환경
                          └── 외부 변수들 (count, name, ...)
```

---

## 정리

- **스코프**: 변수가 유효한 범위. JavaScript는 렉시컬 스코프를 따른다.
- **스코프 체인**: 중첩된 스코프에서 변수를 탐색하는 순서 (안 → 밖).
- **클로저**: 함수가 선언 당시의 외부 변수 환경을 기억하는 것.
- 클로저는 데이터 은닉, 함수 팩토리, 부분 적용 등에 활용된다.
- `var` + 반복문 조합에서의 클로저 함정에 주의해야 한다.
- 불필요한 클로저 참조는 메모리 누수로 이어질 수 있다.
