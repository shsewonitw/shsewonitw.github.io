---
layout: post
title: "[Daily morning study] 웹 성능 최적화 — Lazy Loading과 Code Splitting"
description: >
  #daily morning study
category: 
    - dms
    - dms-frontend
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 왜 웹 성능 최적화가 필요한가

브라우저가 페이지를 처음 로드할 때 모든 JS, CSS, 이미지를 한꺼번에 내려받으면 초기 로딩 시간(TTI, Time to Interactive)이 길어진다. 사용자는 실제로 한 번에 모든 리소스를 필요로 하지 않기 때문에, 필요한 시점에 필요한 리소스만 가져오는 전략이 핵심이다.

주요 성능 지표:

| 지표 | 설명 |
|------|------|
| FCP (First Contentful Paint) | 첫 번째 콘텐츠가 화면에 그려지는 시간 |
| TTI (Time to Interactive) | 페이지가 완전히 상호작용 가능해지는 시간 |
| LCP (Largest Contentful Paint) | 가장 큰 콘텐츠가 렌더링되는 시간 |
| CLS (Cumulative Layout Shift) | 레이아웃이 얼마나 흔들리는지 |

---

## Code Splitting

번들러(Webpack, Vite 등)는 기본적으로 모든 JS를 하나의 파일로 묶는다. 앱이 커질수록 이 파일이 수 MB에 달할 수 있고, 파싱·컴파일 비용이 커진다.

Code Splitting은 번들을 여러 청크(chunk)로 나눠 필요한 청크만 그때그때 로드하는 기법이다.

### Dynamic Import (ES Module)

```js
// 정적 import — 번들 타임에 포함
import { heavyModule } from './heavyModule';

// 동적 import — 런타임에 필요할 때 로드
const { heavyModule } = await import('./heavyModule');
```

### React에서의 Code Splitting — React.lazy + Suspense

```jsx
import React, { Suspense, lazy } from 'react';

const Dashboard = lazy(() => import('./Dashboard'));

function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <Dashboard />
    </Suspense>
  );
}
```

`React.lazy`는 동적 import를 감싸서 컴포넌트가 처음 렌더링될 때 해당 청크를 로드한다. `Suspense`는 청크를 받아오는 동안 보여줄 fallback UI를 지정한다.

### Route-based Code Splitting

가장 효과적인 패턴은 라우트 단위로 청크를 나누는 것이다. 사용자가 해당 페이지에 진입할 때만 JS를 로드한다.

```jsx
// React Router + React.lazy 조합
const Home = lazy(() => import('./pages/Home'));
const About = lazy(() => import('./pages/About'));
const Profile = lazy(() => import('./pages/Profile'));

function App() {
  return (
    <Suspense fallback={<PageSpinner />}>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
        <Route path="/profile" element={<Profile />} />
      </Routes>
    </Suspense>
  );
}
```

### Webpack의 Magic Comments

```js
// 청크 이름 지정
const Dashboard = lazy(() =>
  import(/* webpackChunkName: "dashboard" */ './Dashboard')
);

// 미리 가져오기 (사용자가 곧 필요할 것 같을 때)
import(/* webpackPrefetch: true */ './HeavyFeature');

// 미리 로드 (현재 내비게이션에서 곧 필요)
import(/* webpackPreload: true */ './CriticalModule');
```

`prefetch`는 브라우저 유휴 시간에 미리 다운로드하고, `preload`는 현재 페이지 로드와 병렬로 다운로드한다.

---

## Lazy Loading

### 이미지 Lazy Loading

뷰포트에 들어오기 전까지 이미지를 로드하지 않는 방식. 스크롤이 많은 페이지에서 초기 로드를 크게 줄인다.

**HTML 네이티브 방식 (권장)**

```html
<img src="photo.jpg" loading="lazy" alt="..." />
```

`loading="lazy"` 속성은 모던 브라우저에서 광범위하게 지원된다. 별도 JS 없이 동작한다.

**Intersection Observer API**

네이티브 속성이 지원되지 않거나 더 세밀한 제어가 필요할 때 사용한다.

```js
const images = document.querySelectorAll('img[data-src]');

const observer = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      const img = entry.target;
      img.src = img.dataset.src;
      observer.unobserve(img);
    }
  });
}, {
  rootMargin: '100px', // 뷰포트 100px 전부터 미리 로드
});

images.forEach(img => observer.observe(img));
```

```html
<!-- data-src에 실제 URL을 넣고, src는 placeholder -->
<img data-src="real-photo.jpg" src="placeholder.png" alt="..." />
```

### React에서 이미지 Lazy Loading

```jsx
function LazyImage({ src, alt }) {
  const imgRef = useRef(null);
  const [loaded, setLoaded] = useState(false);

  useEffect(() => {
    const observer = new IntersectionObserver(([entry]) => {
      if (entry.isIntersecting) {
        setLoaded(true);
        observer.disconnect();
      }
    });
    observer.observe(imgRef.current);
    return () => observer.disconnect();
  }, []);

  return (
    <img
      ref={imgRef}
      src={loaded ? src : undefined}
      alt={alt}
      style={{ minHeight: 200 }} // 레이아웃 흔들림 방지
    />
  );
}
```

### 컴포넌트 Lazy Loading

스크롤 아래쪽에 있는 무거운 컴포넌트(지도, 차트, 에디터 등)도 Intersection Observer로 지연 로드할 수 있다.

```jsx
const HeavyChart = lazy(() => import('./HeavyChart'));

function Page() {
  const [visible, setVisible] = useState(false);
  const ref = useRef(null);

  useEffect(() => {
    const observer = new IntersectionObserver(([entry]) => {
      if (entry.isIntersecting) setVisible(true);
    });
    observer.observe(ref.current);
    return () => observer.disconnect();
  }, []);

  return (
    <div ref={ref}>
      {visible && (
        <Suspense fallback={<Spinner />}>
          <HeavyChart />
        </Suspense>
      )}
    </div>
  );
}
```

---

## Tree Shaking

Code Splitting과 함께 알아야 할 최적화 기법. 사용되지 않는 코드를 번들에서 제거한다.

```js
// 전체 import — lodash 전체가 번들에 포함됨
import _ from 'lodash'; // ~70KB

// Named import — tree shaking이 동작하면 필요한 것만 포함
import { debounce } from 'lodash-es'; // 더 작은 크기
```

ES Module 형식(`import/export`)이어야 tree shaking이 제대로 동작한다. CommonJS(`require`)는 정적 분석이 어려워 tree shaking이 잘 안 된다.

---

## Next.js에서의 최적화

Next.js는 기본적으로 라우트 단위 Code Splitting을 자동으로 적용한다.

```jsx
// next/image — 자동 lazy loading, WebP 변환, 크기 최적화
import Image from 'next/image';

function Profile() {
  return (
    <Image
      src="/profile.jpg"
      width={500}
      height={500}
      alt="Profile"
      // loading="lazy" 가 기본값
      // priority 속성을 주면 LCP 대상 이미지는 eager 로드
    />
  );
}

// next/dynamic — React.lazy + Suspense 래퍼
import dynamic from 'next/dynamic';

const HeavyEditor = dynamic(() => import('./HeavyEditor'), {
  loading: () => <p>Loading editor...</p>,
  ssr: false, // 서버사이드 렌더링 제외 (브라우저 전용 모듈일 때)
});
```

---

## 요약

| 기법 | 목적 | 핵심 도구 |
|------|------|----------|
| Code Splitting | 초기 JS 번들 크기 축소 | dynamic import, React.lazy |
| Route-based Splitting | 페이지별 JS 분리 | React Router + lazy, Next.js App Router |
| Lazy Loading (이미지) | 뷰포트 밖 이미지 로드 지연 | `loading="lazy"`, Intersection Observer |
| Lazy Loading (컴포넌트) | 화면 밖 무거운 컴포넌트 로드 지연 | Intersection Observer + Suspense |
| Tree Shaking | 미사용 코드 제거 | ES Module, Webpack/Vite |
| Prefetch/Preload | 다음 페이지 리소스 선제 다운로드 | `<link rel="prefetch">`, webpackPrefetch |

핵심은 **사용자가 당장 필요하지 않은 리소스는 로드하지 않는다**는 원칙이다. Code Splitting으로 JS 청크를 나누고, Lazy Loading으로 이미지와 컴포넌트의 로드 시점을 뒤로 미루면 TTI와 LCP를 크게 개선할 수 있다.
