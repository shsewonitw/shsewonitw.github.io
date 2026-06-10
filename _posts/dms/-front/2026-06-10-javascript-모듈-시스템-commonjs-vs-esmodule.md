---
layout: post
title: "[Daily morning study] JavaScript 모듈 시스템 (CommonJS vs ES Module)"
description: >
  #daily morning study
category: 
    - dms
    - dms-frontend
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 모듈 시스템이 필요한 이유

자바스크립트는 원래 브라우저에서 작은 스크립트를 실행하기 위해 만들어진 언어라 모듈 시스템이 없었다. 파일이 늘어나면서 전역 스코프 오염, 의존성 순서 문제, 코드 재사용 어려움이 생겼고, 이를 해결하기 위해 커뮤니티 주도로 CommonJS, AMD 등이 등장했다. ES2015(ES6)에 이르러 표준 모듈 시스템인 ES Module이 언어 스펙에 포함됐다.

---

## CommonJS (CJS)

Node.js에서 채택한 모듈 시스템. `require()`와 `module.exports`를 사용한다.

```js
// math.js
function add(a, b) { return a + b; }
module.exports = { add };

// app.js
const { add } = require('./math');
console.log(add(1, 2)); // 3
```

**핵심 특징:**

- **동기 로딩**: `require()` 호출 시점에 파일을 즉시 읽고 실행한다. 서버 환경(파일 시스템 접근)에 적합하다.
- **런타임 동적 로딩**: 조건문이나 함수 안에서 `require()`를 호출할 수 있다.
- **모듈 캐싱**: 처음 로드된 모듈은 `require.cache`에 저장되어 이후 동일 경로로 `require()` 하면 캐시를 반환한다.

```js
// 동적 require 예시
function loadConfig(env) {
  return require(`./config/${env}.js`);
}
```

---

## ES Module (ESM)

ES2015에서 언어 표준으로 도입된 모듈 시스템. `import`/`export` 문법을 사용한다.

```js
// math.js
export function add(a, b) { return a + b; }
export const PI = 3.14;

// default export
export default function multiply(a, b) { return a * b; }

// app.js
import multiply, { add, PI } from './math.js';
console.log(add(1, 2)); // 3
console.log(PI);        // 3.14
```

**핵심 특징:**

- **정적 구조**: `import`/`export`는 파일 최상단에서만 선언 가능하다. 파싱 단계에서 의존성 그래프를 파악한다.
- **비동기 로딩**: 브라우저에서 네트워크 요청 없이 병렬로 모듈을 로딩할 수 있다.
- **Tree Shaking 지원**: 정적 분석으로 사용되지 않는 export를 번들에서 제거할 수 있다.
- **브라우저 네이티브 지원**: `<script type="module">`으로 번들러 없이 브라우저에서 직접 사용 가능하다.

---

## CJS vs ESM 비교

| 항목 | CommonJS | ES Module |
|------|----------|-----------|
| 문법 | `require()` / `module.exports` | `import` / `export` |
| 로딩 방식 | 동기 | 비동기 |
| 의존성 분석 | 런타임 | 파싱 타임 (정적) |
| 동적 로딩 | 조건문 안 `require()` 가능 | `import()` 함수 사용 |
| Tree Shaking | 어렵다 | 지원 |
| 브라우저 네이티브 | 불가 (번들러 필요) | 지원 (`type="module"`) |
| 확장자 | `.js`, `.cjs` | `.mjs` 또는 `"type": "module"` 설정 시 `.js` |

---

## 동적 import()

ESM에서 런타임에 모듈을 불러오려면 동적 `import()`를 사용한다. Promise를 반환하므로 `await`와 함께 쓰인다.

```js
// 버튼 클릭 시 모듈 지연 로딩 (코드 스플리팅에 활용)
button.addEventListener('click', async () => {
  const { draw } = await import('./chart.js');
  draw();
});
```

번들러(Webpack, Vite)는 동적 `import()`를 만나면 해당 모듈을 별도의 청크로 분리해 필요할 때만 다운로드하게 한다.

---

## Tree Shaking

ESM의 정적 구조 덕분에 번들러가 실제로 사용하는 export만 번들에 포함시킬 수 있다.

```js
// utils.js
export function used() { return 'used'; }
export function unused() { return 'unused'; }

// app.js
import { used } from './utils.js';
// unused()는 번들에서 제거됨
```

CJS는 `require()`가 런타임에 동적으로 실행되기 때문에 빌드 타임에 어떤 export가 실제 사용되는지 정적으로 파악하기 어려워 Tree Shaking 적용이 제한적이다.

---

## 번들러와의 관계

브라우저는 파일 시스템에 직접 접근할 수 없고, 수백 개의 `import`를 그대로 처리하면 HTTP 요청이 폭증한다. 번들러가 여러 모듈 파일을 하나(또는 여러 개)의 번들로 합쳐 이 문제를 해결한다.

**Webpack**
- CJS, ESM 모두 지원하며 내부적으로 CJS 방식의 런타임을 생성한다
- 코드 스플리팅, Tree Shaking, 다양한 로더/플러그인 생태계

**Vite**
- 개발 서버: ESM 네이티브 활용 → 모듈을 번들링 없이 브라우저에 그대로 전달, HMR이 빠르다
- 프로덕션 빌드: Rollup 기반으로 최적화된 번들 생성
- ESM 중심 설계로 Tree Shaking이 기본 동작

```html
<!-- 번들러 없이 브라우저에서 직접 ESM 사용 -->
<script type="module">
  import { add } from './math.js';
  console.log(add(1, 2));
</script>
```

---

## Node.js에서 ESM 사용하기

Node.js 12+ 부터 ESM을 지원한다. 두 가지 방법으로 활성화한다.

**방법 1**: 파일 확장자 `.mjs` 사용

```js
// utils.mjs
export const greet = (name) => `Hello, ${name}`;
```

**방법 2**: `package.json`에 `"type": "module"` 추가

```json
{
  "type": "module"
}
```

이 경우 `.js` 파일이 ESM으로 처리된다. CJS를 섞어 써야 한다면 해당 파일의 확장자를 `.cjs`로 바꿔야 한다.

---

## 정리

- **CJS**: Node.js 생태계 기반, 동기 로딩, 동적 require 가능, Tree Shaking 어려움
- **ESM**: 언어 표준, 정적 구조, 비동기 로딩, Tree Shaking 지원, 브라우저 네이티브
- 최근 생태계는 ESM으로 전환 중. 라이브러리 배포 시에도 CJS와 ESM을 둘 다 제공하는 **듀얼 패키지** 형태가 흔해졌다.
