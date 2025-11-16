---
layout: post
title: "[Daily morning study] React Server Components (RSC)의 개념과 장점"
description: >
  #daily morning study
category: 
    - dms
    - -front
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---
# React Server Components (RSC)의 개념과 장점

React Server Components(RSC)는 React 애플리케이션의 성능을 최적화하고, 서버 사이드에서 컴포넌트를 렌더링할 수 있도록 도와주는 혁신적인 기능입니다. RSC는 클라이언트와 서버 간의 데이터 전송을 최소화하고, 메모리 사용을 줄이며, 더 나은 사용자 경험을 제공합니다. 이 문서에서는 RSC의 개념과 장점을 살펴보겠습니다.

## 1. React Server Components란?

React Server Components는 서버에서 직접 렌더링되는 React 컴포넌트로, 클라이언트 측에서 실행되는 JavaScript 코드 없이도 데이터를 가져오고 UI를 구성할 수 있게 해줍니다. 일반적으로 React는 클라이언트에서 렌더링되지만, RSC를 통해 서버에서 데이터를 사전 처리하고, 그 결과를 클라이언트에 전송할 수 있습니다.

### 1.1 기본 구조

RSC는 `*.server.js` 파일 확장자를 가진 컴포넌트로 구현되며, 클라이언트와는 별도로 서버에서 실행됩니다. 서버에서 렌더링된 결과는 HTML로 변환되어 클라이언트로 전송됩니다.

### 1.2 동작 방식

1. 서버에서 RSC를 실행하여 필요한 데이터를 가져옵니다.
2. RSC는 가져온 데이터를 사용하여 HTML을 생성합니다.
3. 생성된 HTML이 클라이언트로 전송되고, 클라이언트는 이를 렌더링합니다.

```javascript
// Example of a Server Component
// MyServerComponent.server.js

import React from 'react';

async function fetchData() {
    const response = await fetch('https://api.example.com/data');
    return response.json();
}

const MyServerComponent = async () => {
    const data = await fetchData();
    
    return (
        <div>
            <h1>Data from Server</h1>
            <ul>
                {data.items.map(item => (
                    <li key={item.id}>{item.name}</li>
                ))}
            </ul>
        </div>
    );
};

export default MyServerComponent;
```

## 2. RSC의 장점

### 2.1 성능 최적화

RSC는 서버 측에서 데이터를 미리 가져오고 렌더링하기 때문에 클라이언트는 필요한 HTML을 즉시 받아볼 수 있습니다. 이로 인해 클라이언트의 렌더링 시간이 감소하고, 초기 로딩이 빨라집니다.

### 2.2 데이터 패칭의 간소화

서버에서 컴포넌트를 렌더링할 때 데이터 패칭을 직접적으로 수행할 수 있습니다. 클라이언트에서 추가적인 데이터 요청을 할 필요가 없으므로 요청 수가 줄어들고, 복잡한 상태 관리 없이 UI를 구성할 수 있습니다.

### 2.3 메모리 사용량 감소

클라이언트에서 렌더링해야 할 JavaScript 코드가 줄어들어 메모리 사용량이 감소합니다. 불필요한 코드가 클라이언트로 전송되지 않기 때문에 성능이 향상됩니다.

### 2.4 SEO 향상

서버 측에서 신속하게 렌더링된 HTML을 제공하므로 검색 엔진 최적화(SEO)에도 긍정적인 영향을 미칩니다. 검색 엔진은 완전한 HTML 페이지를 인식할 수 있어 크롤링이 용이해집니다.

### 2.5 코드 분할 및 유지 관리 용이

RSC를 활용하면 서버 전용 코드를 분리할 수 있어, 클라이언트와 서버 관련 코드의 유지 관리가 용이해집니다. 코드의 명확한 경계가 설정되어 가독성이 향상됩니다.

## 3. 주의사항

RSC는 모든 경우에 이상적인 솔루션은 아닙니다. 클라이언트에서 대화형 기능이 많은 애플리케이션에는 완전히 적합하지 않을 수 있습니다. 또한, RSC를 사용하기 위해 기존의 코드 구조를 조정해야 할 수도 있습니다.

## 4. 결론

React Server Components는 서버 렌더링의 이점을 최대한 활용하여 애플리케이션 성능을 개선하고, 사용자 경험을 향상시키는 데 도움을 줍니다. 다양한 장점을 고려할 때, RSC를 활용한 새로운 React 애플리케이션 개발을 고려하는 것이 좋습니다. 필요에 따라 기존 코드와의 통합, 성능 모니터링 등을 통해 최적의 애플리케이션을 구축해 나가길 바랍니다.
