---
layout: post
title: "[Daily morning study] PWA (Progressive Web App) 개념과 구현"
description: >
  #daily morning study
category: 
    - dms
    - dms-frontend
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## PWA란

PWA(Progressive Web App)는 웹 기술(HTML, CSS, JS)로 만들었지만 네이티브 앱처럼 동작하도록 설계된 웹 애플리케이션이다. 홈 화면에 설치, 오프라인 동작, 푸시 알림 등 네이티브 앱의 특성을 웹에서 구현한다.

핵심 목표는 세 가지다.

- **Reliable**: 네트워크가 불안정하거나 오프라인 상태에서도 동작
- **Fast**: 빠른 로딩과 부드러운 애니메이션
- **Engaging**: 홈 화면 설치, 전체 화면 모드, 푸시 알림 등 네이티브 경험

---

## PWA의 세 가지 핵심 기술

### 1. Service Worker

Service Worker는 브라우저와 네트워크 사이에서 동작하는 프록시 스크립트다. 독립적인 스레드에서 실행되므로 DOM에 접근할 수 없고, 페이지가 닫혀도 백그라운드에서 계속 살아 있다.

주요 역할:

- 네트워크 요청 가로채기 및 캐시 응답
- 백그라운드 동기화
- 푸시 알림 수신

### 2. Web App Manifest

앱의 메타 정보를 담은 JSON 파일이다. 브라우저가 이 파일을 읽어 앱 이름, 아이콘, 시작 URL, 화면 방향 등을 결정한다.

### 3. HTTPS

Service Worker는 HTTPS 환경에서만 동작한다. 로컬 개발 시에는 `localhost`도 허용된다.

---

## Service Worker 생명주기

```
페이지 로드 → register() → 다운로드 → 설치(install) → 활성화(activate) → 동작(fetch/push/sync)
```

각 단계별로 이벤트가 발생하고, 이 안에서 캐시 처리를 한다.

```js
// service-worker.js

const CACHE_NAME = 'my-app-v1';
const ASSETS = [
  '/',
  '/index.html',
  '/styles.css',
  '/app.js',
];

// 설치 단계: 정적 자산을 캐시에 저장
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => cache.addAll(ASSETS))
  );
  self.skipWaiting(); // 즉시 활성화
});

// 활성화 단계: 오래된 캐시 정리
self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then((keys) =>
      Promise.all(
        keys
          .filter((key) => key !== CACHE_NAME)
          .map((key) => caches.delete(key))
      )
    )
  );
  self.clients.claim(); // 현재 열려 있는 페이지도 즉시 제어
});

// fetch 이벤트: 네트워크 요청 가로채기
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then((cached) => cached || fetch(event.request))
  );
});
```

---

## 캐싱 전략

| 전략 | 동작 방식 | 적합한 리소스 |
|------|----------|--------------|
| Cache First | 캐시 우선, 없으면 네트워크 | 정적 자산 (이미지, CSS, JS) |
| Network First | 네트워크 우선, 실패 시 캐시 | API 응답, 동적 콘텐츠 |
| Stale While Revalidate | 캐시 즉시 반환 + 백그라운드에서 갱신 | 자주 바뀌지 않는 데이터 |
| Cache Only | 오직 캐시만 사용 | 오프라인 전용 자산 |
| Network Only | 오직 네트워크만 사용 | 인증, 결제 등 실시간 필수 |

**Stale While Revalidate** 예시:

```js
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.open(CACHE_NAME).then((cache) => {
      return cache.match(event.request).then((cached) => {
        const networkFetch = fetch(event.request).then((response) => {
          cache.put(event.request, response.clone());
          return response;
        });
        return cached || networkFetch; // 캐시 있으면 즉시 반환, 동시에 갱신
      });
    })
  );
});
```

---

## Web App Manifest 작성

```json
{
  "name": "My PWA App",
  "short_name": "MyApp",
  "description": "A sample PWA application",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#1976d2",
  "orientation": "portrait",
  "icons": [
    {
      "src": "/icons/icon-192.png",
      "sizes": "192x192",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-512.png",
      "sizes": "512x512",
      "type": "image/png"
    }
  ]
}
```

`display` 값에 따라 보여지는 형태가 달라진다.

| display 값 | 설명 |
|-----------|------|
| `standalone` | 네이티브 앱처럼 브라우저 UI 없이 표시 |
| `fullscreen` | 상태바까지 포함 전체 화면 |
| `minimal-ui` | 최소한의 브라우저 UI (뒤로 가기 버튼 등) |
| `browser` | 일반 브라우저 탭처럼 표시 |

HTML에서는 `<link>`로 연결한다.

```html
<link rel="manifest" href="/manifest.json" />
<meta name="theme-color" content="#1976d2" />
```

---

## Service Worker 등록

```js
// main.js (또는 React의 index.js)

if ('serviceWorker' in navigator) {
  window.addEventListener('load', () => {
    navigator.serviceWorker
      .register('/service-worker.js')
      .then((registration) => {
        console.log('SW registered:', registration.scope);
      })
      .catch((err) => {
        console.error('SW registration failed:', err);
      });
  });
}
```

---

## 오프라인 페이지 구현

네트워크도 없고 캐시도 없을 때 fallback 페이지를 보여주는 패턴이다.

```js
// service-worker.js
const OFFLINE_PAGE = '/offline.html';

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => {
      return cache.addAll([...ASSETS, OFFLINE_PAGE]);
    })
  );
});

self.addEventListener('fetch', (event) => {
  if (event.request.mode === 'navigate') {
    event.respondWith(
      fetch(event.request).catch(() => caches.match(OFFLINE_PAGE))
    );
  }
});
```

---

## 푸시 알림

Service Worker의 `push` 이벤트로 알림을 표시한다. 서버에서 Web Push Protocol을 통해 메시지를 전송한다.

```js
// service-worker.js
self.addEventListener('push', (event) => {
  const data = event.data?.json() ?? { title: '새 알림', body: '내용 없음' };

  event.waitUntil(
    self.registration.showNotification(data.title, {
      body: data.body,
      icon: '/icons/icon-192.png',
    })
  );
});
```

사용자 구독 등록:

```js
const registration = await navigator.serviceWorker.ready;
const subscription = await registration.pushManager.subscribe({
  userVisibleOnly: true,
  applicationServerKey: urlBase64ToUint8Array(PUBLIC_VAPID_KEY),
});
// subscription 객체를 서버에 저장
```

---

## PWA vs 네이티브 앱

| 항목 | PWA | 네이티브 앱 |
|------|-----|-----------|
| 설치 | 브라우저에서 바로 설치 | 앱스토어 심사 필요 |
| 업데이트 | 서버 배포만으로 즉시 반영 | 앱스토어 업데이트 |
| 접근성 | URL 공유 가능 | 앱스토어 링크 필요 |
| 기기 API | 제한적 (카메라, GPS 등 일부) | 모든 기기 API 접근 |
| 성능 | 브라우저 엔진 의존 | 최고 수준 |
| 개발 비용 | 하나의 코드베이스 | iOS/Android 별도 개발 |

---

## Workbox로 간편하게 구현하기

Google의 Workbox 라이브러리를 사용하면 복잡한 Service Worker 로직을 간단히 작성할 수 있다.

```js
import { registerRoute } from 'workbox-routing';
import { CacheFirst, NetworkFirst, StaleWhileRevalidate } from 'workbox-strategies';
import { ExpirationPlugin } from 'workbox-expiration';

// 이미지: Cache First
registerRoute(
  ({ request }) => request.destination === 'image',
  new CacheFirst({
    cacheName: 'images',
    plugins: [new ExpirationPlugin({ maxEntries: 50 })],
  })
);

// API: Network First
registerRoute(
  ({ url }) => url.pathname.startsWith('/api/'),
  new NetworkFirst({ cacheName: 'api-cache' })
);
```

Create React App과 Vite 모두 PWA 플러그인(`vite-plugin-pwa`)을 통해 Workbox 설정을 자동화할 수 있다.

---

## 인스톨 가능 조건 (Chrome 기준)

브라우저가 "홈 화면에 추가" 프롬프트를 띄우려면 다음 조건을 충족해야 한다.

- HTTPS 환경
- `manifest.json`에 `name`, `short_name`, `icons`(192px, 512px), `start_url`, `display` 포함
- Service Worker가 `fetch` 이벤트를 핸들링하고 있어야 함
- 사용자가 사이트를 최소 30초 이상 방문

`beforeinstallprompt` 이벤트를 가로채서 커스텀 설치 버튼을 만들 수도 있다.

```js
let deferredPrompt;

window.addEventListener('beforeinstallprompt', (e) => {
  e.preventDefault();
  deferredPrompt = e;
  showInstallButton(); // 직접 만든 설치 버튼 표시
});

installButton.addEventListener('click', async () => {
  deferredPrompt.prompt();
  const { outcome } = await deferredPrompt.userChoice;
  console.log('설치 결과:', outcome); // 'accepted' | 'dismissed'
  deferredPrompt = null;
});
```
