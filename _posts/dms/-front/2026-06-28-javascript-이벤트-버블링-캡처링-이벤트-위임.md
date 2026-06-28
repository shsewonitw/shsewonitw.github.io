---
layout: post
title: "[Daily morning study] JavaScript 이벤트 버블링(Bubbling), 캡처링(Capturing), 이벤트 위임(Event Delegation)"
description: >
  #daily morning study
category: 
    - dms
    - dms-frontend
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 이벤트 전파(Event Propagation)란

DOM 요소에서 이벤트가 발생하면 해당 요소에서만 끝나는 게 아니라 DOM 트리를 따라 전파된다. 전파 방향에 따라 **캡처링**과 **버블링** 두 단계로 나뉜다.

---

## 이벤트 전파 3단계

| 단계 | 이름 | 방향 |
|------|------|------|
| 1 | 캡처링(Capturing) | document → 이벤트 타깃 방향으로 내려감 |
| 2 | 타깃(Target) | 이벤트가 실제로 발생한 요소 |
| 3 | 버블링(Bubbling) | 이벤트 타깃 → document 방향으로 올라감 |

`addEventListener`는 기본적으로 버블링 단계에서 동작한다.

---

## 이벤트 버블링 (Event Bubbling)

이벤트가 발생한 요소에서 시작해 부모 요소 방향으로 전파되는 것. 대부분의 이벤트는 버블링된다.

```html
<div id="outer">
  <div id="inner">
    <button id="btn">클릭</button>
  </div>
</div>
```

```js
document.getElementById('btn').addEventListener('click', () => console.log('button'));
document.getElementById('inner').addEventListener('click', () => console.log('inner'));
document.getElementById('outer').addEventListener('click', () => console.log('outer'));
```

버튼을 클릭하면:

```
button → inner → outer
```

---

## 이벤트 캡처링 (Event Capturing)

버블링과 반대 방향. 최상위에서 시작해 이벤트 발생 요소 쪽으로 내려온다.

`addEventListener`의 세 번째 인자로 `true`(또는 `{ capture: true }`)를 넘기면 캡처링 단계에서 핸들러가 실행된다.

```js
document.getElementById('outer').addEventListener('click', () => console.log('outer'), true);
document.getElementById('inner').addEventListener('click', () => console.log('inner'), true);
document.getElementById('btn').addEventListener('click', () => console.log('button'), true);
```

버튼을 클릭하면:

```
outer → inner → button
```

---

## 전파 중단 — `stopPropagation`

이벤트 전파를 중간에서 막는다.

```js
document.getElementById('inner').addEventListener('click', (e) => {
  e.stopPropagation(); // 이 이후 outer로 버블링되지 않음
  console.log('inner');
});
```

`stopImmediatePropagation()`은 같은 요소에 등록된 다른 핸들러들도 모두 중단시킨다.

```js
element.addEventListener('click', (e) => {
  e.stopImmediatePropagation();
  console.log('첫 번째 핸들러만 실행됨');
});

element.addEventListener('click', () => {
  console.log('이 핸들러는 실행되지 않음');
});
```

---

## `event.target` vs `event.currentTarget`

| 프로퍼티 | 의미 |
|----------|------|
| `event.target` | 이벤트가 실제로 발생한 요소 (클릭한 바로 그 요소) |
| `event.currentTarget` | 이벤트 핸들러가 등록된 요소 |

```js
document.getElementById('outer').addEventListener('click', (e) => {
  console.log(e.target);        // 실제 클릭된 button#btn
  console.log(e.currentTarget); // 핸들러가 붙은 div#outer
});
```

이벤트 위임을 쓸 때 이 차이가 핵심이다.

---

## 이벤트 위임 (Event Delegation)

자식 요소마다 핸들러를 붙이는 대신, 공통 부모 요소 하나에 핸들러를 붙이고 `event.target`으로 어느 자식에서 발생했는지 판별하는 패턴.

```html
<ul id="list">
  <li data-id="1">아이템 1</li>
  <li data-id="2">아이템 2</li>
  <li data-id="3">아이템 3</li>
</ul>
```

```js
// ❌ 비효율: 각 li에 핸들러 등록
document.querySelectorAll('li').forEach(li => {
  li.addEventListener('click', (e) => {
    console.log(e.target.dataset.id);
  });
});

// ✅ 이벤트 위임: 부모 ul에만 핸들러 등록
document.getElementById('list').addEventListener('click', (e) => {
  if (e.target.tagName === 'LI') {
    console.log(e.target.dataset.id);
  }
});
```

### 이벤트 위임의 장점

1. **메모리 절약**: 핸들러 수가 요소 수에 비례하지 않아도 됨
2. **동적 요소 대응**: 나중에 추가된 자식 요소에도 자동으로 작동
3. **코드 단순화**: 관리 포인트가 하나

---

## 실전 예시 — 동적 목록 처리

```js
const todoList = document.getElementById('todo-list');

function addTodo(text) {
  const li = document.createElement('li');
  li.innerHTML = `<span>${text}</span><button class="delete">삭제</button>`;
  todoList.appendChild(li);
}

// 삭제 버튼은 이벤트 위임으로 처리
todoList.addEventListener('click', (e) => {
  if (e.target.classList.contains('delete')) {
    e.target.parentElement.remove();
  }
});

addTodo('할 일 1');
addTodo('할 일 2');
```

`addTodo()`로 나중에 추가된 항목도 별도 핸들러 없이 삭제가 동작한다.

---

## 버블링되지 않는 이벤트

`focus`, `blur`, `scroll`, `mouseenter`, `mouseleave` 등은 버블링이 일어나지 않는다. 이 경우 이벤트 위임을 쓰려면 버블링 지원 대응 이벤트를 써야 한다.

| 버블링 없음 | 버블링 있는 대안 |
|------------|----------------|
| `focus` | `focusin` |
| `blur` | `focusout` |
| `mouseenter` | `mouseover` |
| `mouseleave` | `mouseout` |

```js
// focus는 버블링 안 됨 → focusin 사용
document.getElementById('form').addEventListener('focusin', (e) => {
  console.log('포커스된 요소:', e.target);
});
```

---

## 정리

- 이벤트는 캡처링(하향) → 타깃 → 버블링(상향) 순으로 전파된다
- `addEventListener` 기본 동작은 버블링 단계
- `stopPropagation()`으로 전파를 중단할 수 있다
- 이벤트 위임은 부모에 핸들러 하나만 등록해 자식 이벤트를 `event.target`으로 처리하는 패턴으로, 성능과 유지보수 측면에서 유리하다
- `focus`, `mouseenter` 등 일부 이벤트는 버블링되지 않으므로 위임 시 주의가 필요하다
