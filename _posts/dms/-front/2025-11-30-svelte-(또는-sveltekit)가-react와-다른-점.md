---
layout: post
title: "[Daily morning study] Svelte (또는 SvelteKit)가 React와 다른 점"
description: >
  #daily morning study
category: 
    - dms
    - -front
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---
# Svelte와 SvelteKit vs. React: 주요 차이점

## 1. 개요

Svelte는 현대 웹 애플리케이션을 구축하기 위한 JavaScript 프레임워크이다. SvelteKit은 Svelte를 기반으로 발전한 애플리케이션 프레임워크로 서버사이드 렌더링(SSR) 및 정적 사이트 생성(SSG)을 지원한다. React는 사용자 인터페이스를 구성하기 위한 JavaScript 라이브러리로, 컴포넌트 기반 아키텍처를 가지고 있다. 이 두 생태계는 서로 다르며, 여기서 Svelte/SvelteKit과 React의 차이점을 알아보자.

## 2. 컴파일 vs. 런타임

- **Svelte/SvelteKit**: Svelte는 컴파일러로, 애플리케이션의 소스 코드를 빌드 단계에서 최적화된 JavaScript 코드로 변환한다. 결과적으로 Svelte 애플리케이션은 런타임에서 가벼운 부담을 덜며, 빠른 성능을 제공합니다.

- **React**: React는 런타임 기반 라이브러리로, 컴포넌트를 생성한 후 가상 DOM에서 변경사항을 관리한다. 이로 인해 애플리케이션이 동작할 때 매번 DOM 트리를 비교하는 오버헤드가 발생할 수 있다.

```jsx
// Svelte 예시
<script>
  let count = 0;
  function increment() {
    count += 1;
  }
</script>

<button on:click={increment}>
  Count: {count}
</button>
```

```jsx
// React 예시
import React, { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);
  
  return (
    <button onClick={() => setCount(count + 1)}>
      Count: {count}
    </button>
  );
}
```

## 3. 상태 관리

- **Svelte/SvelteKit**: Svelte는 반응성을 기본으로 가지고 있다. 변수를 업데이트할 때 자동으로 UI가 업데이트된다. 특별한 상태 관리 라이브러리 없이도 설정이 간편하다.

- **React**: React는 useState와 useEffect hooks를 통해 상태를 관리한다. 복잡한 상태 관리가 필요할 경우 Redux와 같은 외부 상태 관리 툴을 추가해야 할 수 있다.

## 4. 파일 구조와 라우팅

- **SvelteKit**: SvelteKit은 파일 기반 라우팅을 지원한다. src/routes 폴더에 파일을 생성하면 자동으로 라우트가 설정된다. 기본적으로 폴더 구조에 따라 URL이 정의된다.

- **React**: React에서는 React Router와 같은 라이브러리를 사용하여 라우팅을 설정한다. 이 경우, 라우팅을 정의하는 코드가 필요하다.

```plaintext
# SvelteKit 파일 구조 예시
src/
  routes/
    index.svelte       // / 경로
    about.svelte       // /about 경로
```

```jsx
// React Router 예시
import { BrowserRouter as Router, Route, Switch } from 'react-router-dom';
import Home from './Home';
import About from './About';

function App() {
  return (
    <Router>
      <Switch>
        <Route path="/" component={Home} exact />
        <Route path="/about" component={About} />
      </Switch>
    </Router>
  );
}
```

## 5. 스타일링

- **Svelte/SvelteKit**: Svelte는 각 컴포넌트 내에서 CSS를 작성할 수 있도록 지원한다. 스타일은 자동으로 그 컴포넌트에만 적용되며, 스코프가 유지된다.

- **React**: React에서는 CSS-in-JS 라이브러리(Styled-components, Emotion 등)를 사용하거나, 전통적인 CSS파일을 import하여 스타일을 적용할 수 있다.

## 6. 생태계와 커뮤니티

- **Svelte/SvelteKit**: Svelte는 상대적으로 새로운 프레임워크로, 커뮤니티가 빠르게 성장하고 있지만 React에 비해 자료나 리소스가 적을 수 있다.

- **React**: React는 방대한 커뮤니티와 생태계를 갖추고 있다. 많은 라이브러리, 도구, 강력한 공식 문서가 제공된다.

## 7. 결론

Svelte/SvelteKit과 React는 모두 강력한 도구이며, 각각의 장단점이 있다. Svelte는 컴파일 타임에 최적화되고 반응성을 기본으로 제공하여 더 빠른 성능을 낼 수 있다. 반면 React는 방대한 커뮤니티와 유지보수 측면에서 우수한 선택이다. 사용하는 프로젝트의 요구 사항에 맞게 선택하는 것이 중요하다. 각 프레임워크가 제공하는 문서와 커뮤니티 레퍼런스를 통해 더 깊은 이해를 얻을 수 있다.
