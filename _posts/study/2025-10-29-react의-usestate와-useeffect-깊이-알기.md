---
layout: post
title: "[Daily morning study] React의 useState와 useEffect 깊이 알기"
description: >
  #daily morning study
category: 
    - devlog
    - study
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---
# React의 useState와 useEffect 깊이 알기

## 목차

1. [useState 소개](#1-usestate-소개)
2. [useState 사용하기](#2-usestate-사용하기)
3. [useState 실습](#3-usestate-실습)
4. [useEffect 소개](#4-useeffect-소개)
5. [useEffect 사용하기](#5-useeffect-사용하기)
6. [useEffect 실습](#6-useeffect-실습)
7. [정리](#7-정리)

## 1. useState 소개

`useState`는 React에서 상태 관리를 위해 사용하는 가장 기본적인 Hook입니다. 클래스 컴포넌트에서 상태(state)를 관리하기 위해 `this.state`와 `this.setState`를 사용했던 것처럼, 함수 컴포넌트에서는 `useState`를 사용하여 상태 관리를 할 수 있습니다.

### 기본적인 사용법

```javascript
const [state, setState] = useState(initialState);
```

- `useState` 함수는 상태의 초깃값을 인자로 받고, 현재 상태(`state`)와 그 상태를 업데이트하는 함수(`setState`)를 배열로 반환합니다.
- 상태 값은 컴포넌트가 리렌더링되어도 유지됩니다.

## 2. useState 사용하기

컴포넌트에서 `useState`를 사용하는 방법을 보여주는 기본적인 예제입니다.

### 예제 코드

```javascript
import React, { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);

  const increment = () => {
    setCount(count + 1);
  };

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={increment}>
        Click me
      </button>
    </div>
  );
}
```

이 예제에서는 `useState`를 사용하여 `count`라는 상태를 만들고, 이를 업데이트하는 `setCount` 함수를 사용합니다. `increment` 함수는 클릭 시 `count`를 하나씩 증가시키는 기능을 수행합니다.

## 3. useState 실습

1. 새로운 React 프로젝트를 생성하고, 위의 `Counter` 컴포넌트를 추가해보세요.
2. 다양한 상태 값을 관리해보며 `useState`의 사용을 익혀보세요.

## 4. useEffect 소개

`useEffect`는 컴포넌트가 렌더링 될 때마다 특정 작업을 수행하도록 설정할 수 있는 Hook입니다. 클래스 컴포넌트의 생명주기 메소드인 `componentDidMount`, `componentDidUpdate`, `componentWillUnmount`를 합친 형태와 유사합니다.

### 기본적인 사용법

```javascript
useEffect(() => {
  // 코드 작성
}, [dependencies]);
```

- 첫 번째 인자는 실행할 함수입니다.
- 두 번째 인자는 의존성 배열로, 이 배열에 포함된 값들이 변경될 때만 함수가 실행됩니다.

## 5. useEffect 사용하기

컴포넌트에서 API 호출을 수행하는 간단한 예를 들어보겠습니다.

### 예제 코드

```javascript
import React, { useEffect, useState } from 'react';

function ApiFetch() {
  const [data, setData] = useState([]);

  useEffect(() => {
    fetch('https://api.example.com/data')
      .then(response => response.json())
      .then(data => setData(data));
  }, []); // 의존성 배열이 비어 있으면 컴포넌트가 마운트될 때만 실행됩니다.

  return (
    <ul>
      {data.map(item => (
        <li key={item.id}>{item.title}</li>
      ))}
    </ul>
  );
}
```

이 예제에서는 컴포넌트가 마운트 될 때 API를 호출하여 데이터를 가져오고, 이 데이터를 화면에 표시합니다. `useEffect`의 의존성 배열을 비워 컴포넌트가 처음 렌더링될 때만 API 호출이 이루어지도록 합니다.

## 6. useEffect 실습

1. API 호출과 데이터 표시를 수행하는 컴포넌트를 만들어 보세요.
2. 의존성 배열에 다른 상태 값을 추가하여, 해당 상태가 변경될 때마다 API 호출이 이루어지도록 수정해 보세요.

## 7. 정리

`useState`와 `useEffect`는 React의 함수 컴포넌트에서 강력하고 필수적인 도구입니다. `useState`를 통해 상태를 관리하고, `useEffect`로 부수 효과를 처리함으로써, 우리는 더 다이나믹하고 반응이 빠른 웹 애플리케이션을 구축할 수 있습니다. 이 두 가지 Hook의 깊은 이해와 올바른 사용은 현대 웹 개발의 중요한 기술입니다.
