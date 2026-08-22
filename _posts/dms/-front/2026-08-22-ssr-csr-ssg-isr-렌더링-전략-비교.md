---
layout: post
title: "[Daily morning study] SSR vs CSR vs SSG vs ISR 렌더링 전략 비교"
description: >
  #daily morning study
category: 
    - dms
    - dms-frontend
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 렌더링 전략이란

웹 페이지를 어디서, 언제 HTML로 만들어내느냐에 따라 렌더링 전략이 달라진다. 크게 네 가지로 나뉜다.

- **CSR** (Client-Side Rendering)
- **SSR** (Server-Side Rendering)
- **SSG** (Static Site Generation)
- **ISR** (Incremental Static Regeneration)

각 방식은 성능, SEO, 데이터 신선도, 빌드 복잡도 측면에서 트레이드오프가 있다.

---

## CSR — Client-Side Rendering

브라우저가 빈 HTML을 받아서, JavaScript로 DOM을 직접 구성하는 방식이다. React의 기본 동작이 이에 해당한다.

**동작 흐름:**

```
1. 서버 → 빈 HTML + JS 번들 전달
2. 브라우저가 JS 다운로드 & 파싱
3. React가 DOM 생성 (Hydration)
4. 완성된 페이지 노출
```

**장점:**
- 초기 이후 페이지 전환이 빠름 (SPA 경험)
- 서버 부담이 적음
- 동적 인터랙션 구현이 쉬움

**단점:**
- 초기 로딩이 느림 (JS 로드 후에야 콘텐츠 표시)
- SEO에 불리: 검색 엔진이 빈 HTML만 보게 될 수 있음
- 저사양 기기에서 JS 실행 부담

**언제 쓰나:** 로그인 이후 대시보드, 실시간 인터랙션이 많은 웹앱

---

## SSR — Server-Side Rendering

매 요청마다 서버에서 HTML을 완성해서 보내는 방식이다. Next.js의 `getServerSideProps`가 대표적이다.

**동작 흐름:**

```
1. 클라이언트 요청
2. 서버가 데이터 조회 → HTML 생성
3. 완성된 HTML 전달
4. 브라우저가 HTML 표시 + JS Hydration
```

**장점:**
- 초기 로딩 시 완성된 HTML 제공 → SEO에 유리
- 항상 최신 데이터를 렌더링
- 느린 네트워크/기기에서도 콘텐츠를 빠르게 표시

**단점:**
- 요청마다 서버 처리 필요 → 서버 부하 증가
- TTFB(Time To First Byte)가 길어질 수 있음
- 캐싱 전략이 복잡해짐

**언제 쓰나:** 로그인 상태에 따라 다른 콘텐츠, 실시간 데이터가 중요한 페이지 (뉴스, 커머스 상품 상세)

---

## SSG — Static Site Generation

빌드 타임에 미리 HTML을 생성해두고 CDN에 배포하는 방식이다. Next.js의 `getStaticProps`가 해당한다.

**동작 흐름:**

```
1. 빌드 시점에 데이터 조회 → HTML 파일 생성
2. 완성된 HTML을 CDN에 업로드
3. 클라이언트 요청 → CDN이 HTML 즉시 응답
```

**장점:**
- TTFB 최소: CDN에서 바로 파일 제공
- SEO 완벽 지원
- 서버 부하 없음
- 보안상 공격 표면이 적음

**단점:**
- 데이터가 업데이트되면 전체 재빌드 필요
- 페이지 수가 많으면 빌드 시간이 길어짐
- 동적 데이터를 다루기 어려움

**언제 쓰나:** 마케팅 페이지, 문서 사이트, 블로그 (데이터가 자주 바뀌지 않는 경우)

---

## ISR — Incremental Static Regeneration

SSG의 단점(재빌드 필요)을 보완한 방식이다. Next.js에서 `revalidate` 옵션으로 구현한다.

**동작 흐름:**

```
1. 빌드 시점에 HTML 생성 (SSG와 동일)
2. 클라이언트 요청 → 캐싱된 HTML 즉시 응답
3. revalidate 시간이 지난 후 요청이 들어오면:
   → 백그라운드에서 페이지 재생성
   → 다음 요청부터 새 HTML 제공
```

**예시 코드 (Next.js):**

```javascript
export async function getStaticProps() {
  const data = await fetchData();
  return {
    props: { data },
    revalidate: 60, // 60초마다 백그라운드 재생성
  };
}
```

**장점:**
- SSG의 성능(CDN 캐시)을 유지하면서 데이터 신선도 확보
- 전체 재빌드 불필요
- 트래픽 많아도 서버 부하 최소화

**단점:**
- 최신 데이터가 즉시 반영되지 않음 (stale-while-revalidate 패턴)
- 복잡한 데이터 의존성 처리가 까다로움

**언제 쓰나:** 제품 목록, 블로그 포스트 (자주 바뀌지만 즉시성이 필요 없는 경우)

---

## 비교 요약

| 항목 | CSR | SSR | SSG | ISR |
|------|-----|-----|-----|-----|
| HTML 생성 시점 | 브라우저 | 요청마다 서버 | 빌드 시 | 빌드 시 + 주기적 갱신 |
| SEO | ❌ 불리 | ✅ 유리 | ✅ 유리 | ✅ 유리 |
| 초기 로딩 속도 | 느림 | 중간 | 빠름 | 빠름 |
| 데이터 신선도 | 실시간 | 실시간 | 빌드 시점 | 주기적 갱신 |
| 서버 부하 | 낮음 | 높음 | 없음 | 낮음 |
| 대표 사례 | 대시보드 | 뉴스, 커머스 | 블로그, 문서 | 제품 목록 |

---

## Next.js App Router에서의 변화

Next.js 13+ App Router에서는 `getServerSideProps`, `getStaticProps` 대신 컴포넌트 레벨에서 렌더링 전략을 결정한다.

```javascript
// SSR — 동적 렌더링 (매 요청마다 서버에서 실행)
async function Page() {
  const data = await fetch('https://api.example.com/data', {
    cache: 'no-store' // SSR
  });
  ...
}

// SSG — 정적 렌더링 (빌드 시 한 번만)
async function Page() {
  const data = await fetch('https://api.example.com/data', {
    cache: 'force-cache' // SSG (기본값)
  });
  ...
}

// ISR — 주기적 재생성
async function Page() {
  const data = await fetch('https://api.example.com/data', {
    next: { revalidate: 60 } // ISR
  });
  ...
}
```

---

## 실무 선택 기준

1. **SEO가 중요하고 데이터가 거의 안 바뀌면** → SSG
2. **SEO가 중요하고 데이터가 자주 바뀌지만 즉시성은 불필요하면** → ISR
3. **SEO가 중요하고 요청마다 최신 데이터 필수이면** → SSR
4. **SEO 불필요하고 인터랙션 중심이면** → CSR

하나의 앱 안에서도 페이지마다 전략을 다르게 가져갈 수 있다. 메인 랜딩 페이지는 SSG, 뉴스 피드는 ISR, 사용자 프로필은 SSR, 내부 대시보드는 CSR 식으로 조합하는 게 일반적이다.
