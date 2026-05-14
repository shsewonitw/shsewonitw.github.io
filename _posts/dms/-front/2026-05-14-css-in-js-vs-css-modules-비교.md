---
layout: post
title: "[Daily morning study] CSS-in-JS vs CSS Modules 비교"
description: >
  #daily morning study
category: 
    - dms
    - dms-frontend
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## CSS 스타일링 방식의 배경

전통적인 CSS는 전역 스코프로 동작한다. 클래스 이름이 프로젝트 어디서든 충돌할 수 있고, 어떤 컴포넌트가 어떤 스타일을 쓰는지 추적하기 어렵다. 이 문제를 해결하기 위해 **CSS Modules**와 **CSS-in-JS** 두 가지 접근법이 등장했다.

---

## CSS Modules

### 개념

CSS 파일을 모듈처럼 import해서 클래스 이름을 로컬 스코프로 만드는 방식이다. Webpack, Vite 등 주요 빌드 도구에서 기본 지원하며, 별도 런타임이 필요 없다.

### 동작 방식

빌드 타임에 각 클래스명을 고유한 해시가 포함된 이름으로 변환한다.

```css
/* Button.module.css */
.button {
  background-color: blue;
  color: white;
  padding: 8px 16px;
}
```

```jsx
// Button.jsx
import styles from './Button.module.css';

function Button({ children }) {
  return <button className={styles.button}>{children}</button>;
}
```

빌드 후 실제 HTML에는 `.Button_button__abc12` 같은 고유 클래스명이 삽입된다. 충돌이 구조적으로 불가능하다.

### 장점

- 빌드 타임 처리 → 런타임 오버헤드 없음
- 기존 CSS 문법 그대로 사용 가능
- Sass, Less 등 전처리기와 조합 가능
- React Server Components(RSC)와 완전 호환

### 단점

- 동적 스타일링이 번거롭다. props에 따라 스타일을 바꾸려면 className을 조건부로 조합해야 한다.
- 스타일 파일과 컴포넌트 파일이 분리되어 있어 파일 수가 늘어난다.

```jsx
// 동적 스타일링 — 불편하지만 가능
import styles from './Button.module.css';
import cx from 'classnames';

function Button({ primary, children }) {
  return (
    <button className={cx(styles.button, { [styles.primary]: primary })}>
      {children}
    </button>
  );
}
```

---

## CSS-in-JS

### 개념

스타일을 JS/TSX 파일 내부에서 직접 선언하는 방식이다. 대표 라이브러리로 **styled-components**, **Emotion**이 있다.

### 동작 방식

컴포넌트 단위로 스타일을 선언하고, 런타임에 동적으로 CSS 클래스를 생성해서 `<style>` 태그에 주입한다.

```jsx
// styled-components 예시
import styled from 'styled-components';

const Button = styled.button`
  background-color: ${(props) => (props.$primary ? 'blue' : 'white')};
  color: ${(props) => (props.$primary ? 'white' : 'black')};
  padding: 8px 16px;
`;

function App() {
  return <Button $primary>Click me</Button>;
}
```

Emotion은 `css` prop 방식도 제공한다.

```jsx
/** @jsxImportSource @emotion/react */
import { css } from '@emotion/react';

const buttonStyle = (primary) => css`
  background-color: ${primary ? 'blue' : 'white'};
  color: ${primary ? 'white' : 'black'};
  padding: 8px 16px;
`;

function Button({ primary, children }) {
  return <button css={buttonStyle(primary)}>{children}</button>;
}
```

### 장점

- JS 변수, props, 상태 값을 스타일에 직접 활용 → 동적 스타일링이 자연스럽다
- 스타일과 컴포넌트 로직이 한 파일에 있어 응집도가 높다
- 클래스명 충돌이 자동으로 방지된다

### 단점

- **런타임 오버헤드**: 스타일을 런타임에 생성·주입하기 때문에 성능 비용이 있다
- 라이브러리 자체의 번들 크기 추가 (styled-components ~30KB gzip 기준)
- React Server Components(RSC)에서 동작하지 않는다. 서버 컴포넌트는 런타임 JS를 실행하지 않기 때문이다.
- SSR 환경에서 별도 설정이 필요하다

---

## 비교 요약

| 항목 | CSS Modules | CSS-in-JS (styled-components, Emotion) |
|------|-------------|----------------------------------------|
| 스타일 위치 | 별도 `.module.css` 파일 | JS/TSX 파일 내부 |
| 처리 시점 | 빌드 타임 | 런타임 |
| 동적 스타일링 | 불편 (className 조합 필요) | 자연스럽고 간결함 |
| 런타임 성능 | 우수 (오버헤드 없음) | 상대적으로 느림 |
| RSC 호환성 | 가능 | 대부분 불가 |
| 번들 크기 | 영향 없음 | 라이브러리만큼 증가 |
| CSS 전처리기 지원 | 가능 | 별도 설정 필요 |

---

## Zero-runtime CSS-in-JS

런타임 비용 없이 CSS-in-JS의 편의성을 얻으려는 시도로 **Zero-runtime** 방식이 등장했다. 대표 라이브러리: **vanilla-extract**, **Panda CSS**, **Linaria**

이 도구들은 JS에서 스타일을 작성하지만, 빌드 타임에 정적 CSS 파일을 생성한다.

```ts
// vanilla-extract 예시
import { style } from '@vanilla-extract/css';

export const button = style({
  backgroundColor: 'blue',
  color: 'white',
  padding: '8px 16px',
});
```

```ts
// 동적 스타일은 styleVariants로 처리
import { styleVariants } from '@vanilla-extract/css';

export const buttonVariant = styleVariants({
  primary: { backgroundColor: 'blue', color: 'white' },
  secondary: { backgroundColor: 'white', color: 'black' },
});
```

빌드 후 `.css` 파일이 생성되며, 런타임에는 클래스명만 참조한다. RSC와도 호환되고 성능은 CSS Modules 수준이다.

---

## 선택 기준

- **정적 스타일 위주, 성능 중요, Next.js App Router 사용** → CSS Modules 또는 vanilla-extract
- **동적 스타일링 빈번, 개발 편의성 우선, Pages Router 또는 CRA** → Emotion, styled-components
- **타입 안전성과 Zero-runtime 둘 다 원함** → vanilla-extract

Next.js 13+ App Router에서 런타임 CSS-in-JS를 쓰면 서버 컴포넌트에서는 스타일이 적용되지 않고, 무조건 클라이언트 컴포넌트(`'use client'`)로 선언해야 한다. 이 제약 때문에 최근 프로젝트에서는 CSS Modules나 Tailwind CSS로 전환하는 사례가 늘고 있다.
