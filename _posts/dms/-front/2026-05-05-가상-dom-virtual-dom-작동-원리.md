---
layout: post
title: "[Daily morning study] 가상 DOM (Virtual DOM) 작동 원리"
description: >
  #daily morning study
category: 
    - dms
    - dms-frontend
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 가상 DOM이란

브라우저의 실제 DOM(Document Object Model)을 직접 조작하는 건 느리다. DOM 조작이 발생할 때마다 브라우저는 레이아웃 계산, 페인트, 컴포지팅 과정을 다시 수행하기 때문이다.

가상 DOM(Virtual DOM, VDOM)은 실제 DOM의 경량화된 자바스크립트 복사본이다. 메모리 상에 존재하는 객체 트리로, 실제 DOM 변경 전에 어떤 부분이 바뀌었는지 비교하는 데 사용된다. React가 대표적으로 이 방식을 채택했다.

---

## 실제 DOM의 문제점

브라우저 DOM API는 C++로 구현된 네이티브 객체다. 자바스크립트에서 DOM을 건드리면 JS 엔진과 렌더링 엔진 사이의 경계를 넘어야 하므로 비용이 크다.

```js
// 이 작업 하나하나가 렌더링 파이프라인을 트리거할 수 있다
document.getElementById('count').textContent = newCount;
document.getElementById('list').innerHTML = newHTML;
```

특히 상태 변경이 많은 SPA(Single Page Application)에서는 매번 전체 DOM을 다시 그리면 성능이 급격히 저하된다.

---

## 가상 DOM의 동작 흐름

가상 DOM은 크게 세 단계로 작동한다.

### 1. 가상 DOM 트리 생성

UI를 표현하는 자바스크립트 객체 트리를 만든다. React에서 JSX가 컴파일되면 `React.createElement()` 호출로 변환되고, 이 함수가 가상 DOM 노드(React Element)를 반환한다.

```jsx
// JSX
const element = <div className="box"><p>Hello</p></div>;

// 컴파일 결과 (React.createElement)
const element = React.createElement(
  'div',
  { className: 'box' },
  React.createElement('p', null, 'Hello')
);

// 생성되는 객체 구조
{
  type: 'div',
  props: {
    className: 'box',
    children: {
      type: 'p',
      props: { children: 'Hello' }
    }
  }
}
```

### 2. Diffing (재조정, Reconciliation)

상태나 props가 변경되면 새로운 가상 DOM 트리를 생성한다. 그리고 이전 트리와 새 트리를 비교(diff)해서 실제로 바뀐 부분만 찾아낸다.

순수하게 두 트리를 비교하면 O(n³) 복잡도가 나오지만, React는 두 가지 휴리스틱을 적용해 O(n)으로 줄인다.

| 휴리스틱 | 내용 |
|----------|------|
| 타입이 다르면 전체 교체 | `<div>`가 `<span>`으로 바뀌면 서브트리 전체를 새로 만든다 |
| 같은 타입이면 속성만 비교 | DOM 노드를 재사용하고 변경된 attribute만 업데이트한다 |
| `key`로 리스트 추적 | 리스트 항목의 key가 같으면 동일 노드로 인식해 이동/수정만 처리한다 |

### 3. Patch (실제 DOM 반영)

diff 결과로 나온 변경 사항만 실제 DOM에 일괄 적용(batch update)한다. 변경이 없는 노드는 건드리지 않는다.

```
이전 VDOM          새 VDOM           실제 DOM 변경
─────────         ─────────         ─────────────────
<ul>              <ul>
  <li>A</li>        <li>A</li>       → 변경 없음
  <li>B</li>        <li>B</li>       → 변경 없음
                    <li>C</li>       → <li>C</li> 추가
```

---

## key props가 중요한 이유

리스트를 렌더링할 때 `key`를 제대로 넣지 않으면 diff 알고리즘이 노드를 잘못 추적한다.

```jsx
// key 없음 → 순서 변경 시 전체 재렌더링 가능
items.map(item => <li>{item.name}</li>)

// key 있음 → id로 동일 노드 추적
items.map(item => <li key={item.id}>{item.name}</li>)
```

배열의 인덱스를 key로 쓰면 안 되는 경우: 항목이 추가·삭제·재정렬될 때 인덱스가 바뀌어 React가 노드를 잘못 재사용한다.

```jsx
// 나쁜 예 (항목 순서가 바뀌는 경우)
items.map((item, index) => <li key={index}>{item.name}</li>)

// 좋은 예
items.map(item => <li key={item.id}>{item.name}</li>)
```

---

## React Fiber

React 16부터 기존 재조정 알고리즘을 **Fiber**라는 새 아키텍처로 교체했다. 기존 스택 기반 재조정은 동기적으로 전체 트리를 처리해 중간에 멈출 수 없었다.

Fiber는 재조정 작업을 작은 단위(fiber node)로 쪼개 우선순위에 따라 스케줄링한다.

| 특징 | 내용 |
|------|------|
| 작업 분할 | 렌더링을 조각내어 프레임 사이에 나눠서 처리 |
| 우선순위 | 사용자 입력 > 애니메이션 > 데이터 패칭 순으로 처리 |
| 중단/재개 | 더 급한 작업이 생기면 현재 렌더링을 멈추고 나중에 재개 |
| Concurrent Mode | 이를 기반으로 `Suspense`, `useTransition` 등이 가능해짐 |

---

## 가상 DOM이 항상 빠른가

가상 DOM이 실제 DOM보다 무조건 빠른 건 아니다. 단순 정적 페이지라면 오히려 diff 연산 오버헤드가 추가 비용이 된다.

가상 DOM의 진짜 장점은 "충분히 빠른 성능"을 유지하면서 선언적(declarative) UI 코드를 작성할 수 있다는 데 있다. 개발자가 "어떻게 DOM을 조작할지" 대신 "UI가 어떤 상태여야 하는지"만 기술하면 된다.

Svelte 같은 프레임워크는 가상 DOM을 아예 사용하지 않고 빌드 타임에 DOM 조작 코드를 직접 생성해 런타임 오버헤드를 없앤다. 각 접근법은 트레이드오프가 있다.

---

## 요약

- 가상 DOM은 실제 DOM의 JS 객체 복사본으로, 변경 전 diff를 통해 최소한의 DOM 조작만 수행한다
- React의 diff 알고리즘은 두 가지 휴리스틱으로 O(n³)을 O(n)으로 줄인다
- `key`는 리스트 diff의 정확성에 필수이며, 인덱스를 key로 쓰는 건 주의가 필요하다
- React 16의 Fiber는 재조정 작업을 쪼개 우선순위 기반 스케줄링을 가능하게 한다
- 가상 DOM의 핵심 가치는 절대적인 성능이 아닌, 선언적 UI 프로그래밍 모델이다
