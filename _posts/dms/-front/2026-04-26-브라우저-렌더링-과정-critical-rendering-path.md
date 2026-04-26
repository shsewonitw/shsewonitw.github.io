---
layout: post
title: "[Daily morning study] 브라우저 렌더링 과정 (Critical Rendering Path)"
description: >
  #daily morning study
category: 
    - dms
    - dms-frontend
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 브라우저 렌더링 과정이란?

브라우저가 HTML 문서를 받아서 화면에 픽셀로 그려내기까지의 전 과정을 **Critical Rendering Path(CRP)**라고 한다. 이 과정을 이해하면 왜 특정 코드가 렌더링을 블로킹하는지, 어떻게 최적화할 수 있는지 명확하게 보인다.

---

## 전체 흐름 요약

```
HTML 파싱 → DOM 생성
CSS 파싱 → CSSOM 생성
DOM + CSSOM → Render Tree 생성
Layout (Reflow)
Paint
Composite
```

---

## 1단계: DOM 생성 (HTML 파싱)

브라우저는 서버에서 HTML을 바이트 스트림으로 받는다. 이걸 문자 → 토큰 → 노드 → DOM 트리 순서로 변환한다.

```
Bytes → Characters → Tokens → Nodes → DOM
```

- `<html>`, `<head>`, `<body>` 같은 각 태그가 노드가 된다.
- DOM은 JavaScript가 페이지 콘텐츠를 조작할 수 있게 해주는 트리 구조다.

**파싱 도중 `<script>` 태그를 만나면?**  
기본적으로 HTML 파싱이 중단된다. 스크립트가 DOM을 변경할 수 있기 때문에 브라우저는 이 시점에 JS 실행을 먼저 완료한다. 이게 바로 스크립트가 렌더링을 **블로킹**하는 이유다.

---

## 2단계: CSSOM 생성 (CSS 파싱)

CSS 파일도 DOM과 유사한 방식으로 파싱되어 **CSSOM(CSS Object Model)** 트리가 만들어진다.

- CSSOM은 스타일 규칙과 각 규칙이 어떤 요소에 적용되는지를 표현한다.
- CSS는 **렌더 블로킹 리소스**다. CSSOM이 완성되기 전까지는 Render Tree를 만들 수 없다.
- CSS 파일이 다운로드되지 않으면 페이지 렌더링 자체가 멈춘다.

---

## 3단계: Render Tree 생성

DOM과 CSSOM이 모두 준비되면 이 둘을 합쳐 **Render Tree**를 만든다.

- `display: none`인 요소는 Render Tree에 포함되지 않는다. (공간을 차지하지 않음)
- `visibility: hidden`은 포함된다. (공간은 차지, 화면에 안 보임)
- `<head>`, `<script>`, `<meta>` 같은 비시각적 노드도 제외된다.

```
DOM Tree + CSSOM Tree → Render Tree (화면에 보이는 요소만)
```

---

## 4단계: Layout (Reflow)

Render Tree의 각 노드가 화면에서 **어디에, 얼마나 크게** 위치할지를 계산하는 단계다.

- 뷰포트 크기를 기준으로 모든 요소의 정확한 위치와 크기를 계산한다.
- 이 과정을 **Reflow** 또는 **Layout**이라고도 한다.
- 비용이 큰 작업이다. 요소의 크기나 위치가 바뀌면 다시 계산한다.

**Reflow를 유발하는 CSS 속성 예시:**

| 속성 | 영향 |
|------|------|
| `width`, `height` | 크기 변경 |
| `margin`, `padding` | 간격 변경 |
| `top`, `left` | 위치 변경 |
| `font-size` | 텍스트 크기 변경 |

---

## 5단계: Paint

Layout이 끝나면 실제로 픽셀을 칠하는 단계다. 텍스트, 색상, 이미지, 선, 그림자 등 시각적 요소를 레이어에 그린다.

- 이 단계를 **Repaint**라고도 한다.
- `background-color`, `color`, `border-radius` 등 시각적 속성이 변경되면 Repaint가 발생한다.
- Reflow는 항상 Repaint를 동반하지만, Repaint가 항상 Reflow를 유발하지는 않는다.

---

## 6단계: Composite

그려진 레이어들을 합성해서 최종 화면을 완성하는 단계다.

- 브라우저는 성능을 위해 페이지를 여러 레이어로 나눈다.
- `transform`, `opacity` 같은 속성은 Reflow/Repaint 없이 Composite 단계만 발생시켜서 가장 빠르다.

```
Reflow → Repaint → Composite  (가장 비싼 경로)
Repaint → Composite            (중간)
Composite만                    (가장 빠른 경로)
```

---

## 렌더링 블로킹 리소스

**블로킹 리소스란?** 브라우저가 이 리소스를 처리하기 전까지 렌더링을 진행하지 못하는 것.

| 리소스 | 블로킹 여부 | 이유 |
|--------|-------------|------|
| CSS (`<link>`) | 렌더 블로킹 | CSSOM 완성 전까지 Render Tree 못 만듦 |
| JS (`<script>`) | 파싱 블로킹 | 스크립트 실행 동안 HTML 파싱 중단 |

### 블로킹 해결 방법

**async와 defer 비교:**

```html
<!-- 일반 script: 파싱 중단 후 즉시 실행 -->
<script src="app.js"></script>

<!-- async: 다운로드는 병렬, 완료 즉시 실행 (파싱 중단될 수 있음) -->
<script async src="app.js"></script>

<!-- defer: 다운로드는 병렬, HTML 파싱 완료 후 실행 -->
<script defer src="app.js"></script>
```

- `defer`는 DOM이 완전히 만들어진 후 실행되므로 대부분의 경우 가장 안전하다.
- `async`는 실행 순서가 보장되지 않아 독립적인 스크립트에만 적합하다.

**CSS 블로킹 최소화:**

```html
<!-- 미디어 쿼리로 조건부 로드: 해당 조건 아닐 때 렌더 블로킹 안 함 -->
<link rel="stylesheet" href="print.css" media="print">
<link rel="stylesheet" href="mobile.css" media="(max-width: 768px)">
```

---

## JavaScript와 렌더링의 관계

JS는 DOM과 CSSOM을 모두 수정할 수 있다. 그래서 브라우저는 스크립트를 만나면:

1. HTML 파싱 중단
2. 해당 시점까지 완성된 CSSOM이 있으면 대기 (CSSOM이 JS 실행보다 먼저 완성돼야 함)
3. JS 실행
4. HTML 파싱 재개

이 흐름 때문에 JS를 `<body>` 맨 아래에 두거나 `defer`를 사용하는 것이 권장된다.

---

## 성능 최적화 핵심 포인트

**1. CSS는 최대한 일찍, JS는 최대한 늦게**

```html
<head>
  <link rel="stylesheet" href="styles.css"> <!-- head에 배치 -->
</head>
<body>
  <!-- 콘텐츠 -->
  <script defer src="app.js"></script> <!-- body 끝 또는 defer -->
</body>
```

**2. Reflow를 유발하는 JS 패턴 피하기**

```javascript
// 나쁜 예: 읽고 쓰기가 반복되면서 강제 Reflow 발생
for (let i = 0; i < 100; i++) {
  element.style.left = element.offsetLeft + 1 + 'px'; // 읽기 → 쓰기 반복
}

// 좋은 예: 읽기를 먼저 모아서 처리
const left = element.offsetLeft;
for (let i = 0; i < 100; i++) {
  element.style.left = left + i + 'px';
}
```

**3. transform/opacity 활용으로 Composite만 발생시키기**

```css
/* 나쁜 예: Reflow + Repaint 유발 */
.box {
  left: 100px;
  transition: left 0.3s;
}

/* 좋은 예: Composite만 발생 (GPU 가속) */
.box {
  transform: translateX(100px);
  transition: transform 0.3s;
}
```

**4. will-change로 레이어 분리 힌트 주기**

```css
.animated-element {
  will-change: transform; /* 브라우저에게 해당 레이어 분리 힌트 */
}
```

단, `will-change`를 과도하게 사용하면 메모리를 많이 차지하므로 애니메이션이 실제로 있는 요소에만 적용해야 한다.

---

## 정리

| 단계 | 설명 |
|------|------|
| DOM 생성 | HTML 파싱 → 트리 구조 |
| CSSOM 생성 | CSS 파싱 → 스타일 트리 |
| Render Tree | DOM + CSSOM → 화면에 보이는 요소만 |
| Layout | 각 요소의 위치와 크기 계산 |
| Paint | 픽셀 색칠 |
| Composite | 레이어 합성 → 최종 화면 |

CRP를 최적화하는 핵심은 **렌더 블로킹 리소스를 줄이고**, **Reflow/Repaint 횟수를 최소화**하며, **가능하면 Composite 단계만 발생시키는 것**이다.
