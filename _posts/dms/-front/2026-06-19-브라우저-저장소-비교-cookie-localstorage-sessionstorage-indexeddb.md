---
layout: post
title: "[Daily morning study] 브라우저 저장소 비교 (Cookie, localStorage, sessionStorage, IndexedDB)"
description: >
  #daily morning study
category: 
    - dms
    - dms-frontend
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 브라우저 저장소가 필요한 이유

HTTP는 상태를 유지하지 않는 프로토콜이라, 페이지를 새로고침하거나 다른 페이지로 이동하면 기존 상태가 사라진다. 사용자 로그인 상태, 장바구니 목록, 설정값 같은 데이터를 유지하려면 브라우저 어딘가에 저장해야 한다.

브라우저가 제공하는 클라이언트 측 저장소는 크게 네 가지다.

- **Cookie**: 서버와 데이터를 주고받는 전통적인 방식
- **localStorage**: 탭·브라우저를 닫아도 남는 영구 저장소
- **sessionStorage**: 현재 탭(세션)이 열려있는 동안만 유지
- **IndexedDB**: 대용량 구조화 데이터를 위한 비동기 DB

---

## Cookie

### 특징

- 모든 HTTP 요청 헤더에 자동으로 서버로 전송된다.
- 용량 한도는 약 4KB.
- `expires` / `max-age` 속성으로 만료 시간을 지정할 수 있다.
- 지정하지 않으면 브라우저 세션이 끝날 때(탭/브라우저 종료 시) 삭제된다.

### 보안 속성

| 속성 | 효과 |
| --- | --- |
| `HttpOnly` | JavaScript에서 접근 불가 → XSS 방어 |
| `Secure` | HTTPS 연결에서만 전송 |
| `SameSite=Strict` | 크로스 사이트 요청에서 쿠키 미전송 → CSRF 방어 |
| `SameSite=Lax` | 일부 크로스 사이트 GET 요청에서는 전송 |

### 사용 예

```js
// 쓰기 (7일 만료)
document.cookie = "token=abc123; max-age=604800; Secure; SameSite=Strict";

// 읽기 (직접 파싱해야 해서 불편함)
const cookies = Object.fromEntries(
  document.cookie.split("; ").map(c => c.split("="))
);
console.log(cookies.token); // "abc123"
```

`HttpOnly` 쿠키는 `document.cookie`로 읽을 수 없으므로, 인증 토큰은 서버에서 `HttpOnly`로 세팅하는 것이 안전하다.

---

## localStorage

### 특징

- 같은 오리진(프로토콜 + 도메인 + 포트) 내 모든 탭에서 공유된다.
- 브라우저를 닫아도 데이터가 남는다 — 명시적으로 삭제하기 전까지 영구 보존.
- 용량 한도는 약 5~10MB (브라우저마다 다름).
- HTTP 요청에 자동으로 포함되지 않는다.
- 값은 **문자열**만 저장된다. 객체는 JSON으로 직렬화해야 한다.

### 사용 예

```js
// 저장
localStorage.setItem("theme", "dark");
localStorage.setItem("user", JSON.stringify({ name: "Alice", role: "admin" }));

// 읽기
const theme = localStorage.getItem("theme"); // "dark"
const user = JSON.parse(localStorage.getItem("user")); // { name: "Alice", role: "admin" }

// 삭제
localStorage.removeItem("theme");

// 전체 삭제
localStorage.clear();
```

### 주의점

- 동기 API라 메인 스레드를 블로킹한다. 큰 데이터를 자주 읽고 쓰면 성능 문제가 생길 수 있다.
- JavaScript에서 접근 가능하므로 XSS에 취약하다. 인증 토큰을 여기에 저장하는 건 보안상 좋지 않다.

---

## sessionStorage

### 특징

- **탭 단위**로 격리된다. 같은 도메인이라도 다른 탭과 공유되지 않는다.
- 탭을 닫으면 데이터가 삭제된다.
- API는 localStorage와 동일하다. `localStorage` → `sessionStorage`로만 바꾸면 된다.
- 용량 한도, 문자열 저장 제한도 localStorage와 같다.

### 사용 예

```js
// 단계별 폼 데이터를 탭 내에서만 유지
sessionStorage.setItem("step1", JSON.stringify({ name: "Bob", age: 25 }));

// 탭을 닫으면 자동으로 사라짐
```

### localStorage vs sessionStorage 선택 기준

| 상황 | 선택 |
| --- | --- |
| 재방문 시에도 설정 유지 (테마, 언어) | `localStorage` |
| 로그인 세션 동안만 필요한 임시 상태 | `sessionStorage` |
| 여러 탭이 같은 상태를 공유해야 함 | `localStorage` |
| 탭 간 격리가 필요한 폼 위자드 | `sessionStorage` |

---

## IndexedDB

### 특징

- 브라우저 내장 **NoSQL 객체 스토어**. 수십~수백 MB 이상의 대용량 데이터를 다룰 수 있다.
- JavaScript 객체를 그대로 저장할 수 있고, 인덱스를 걸어 빠르게 조회한다.
- **비동기 API**라 메인 스레드를 블로킹하지 않는다.
- 같은 오리진 내에서 공유, 브라우저를 닫아도 데이터 유지.
- 트랜잭션 기반으로 ACID 속성을 보장한다.

### 사용 예 (기본 API)

```js
const request = indexedDB.open("myDB", 1);

request.onupgradeneeded = (event) => {
  const db = event.target.result;
  const store = db.createObjectStore("todos", { keyPath: "id", autoIncrement: true });
  store.createIndex("by_status", "status", { unique: false });
};

request.onsuccess = (event) => {
  const db = event.target.result;

  // 쓰기
  const tx = db.transaction("todos", "readwrite");
  tx.objectStore("todos").add({ title: "IndexedDB 공부", status: "done" });

  // 읽기
  const readTx = db.transaction("todos", "readonly");
  const getReq = readTx.objectStore("todos").get(1);
  getReq.onsuccess = () => console.log(getReq.result);
};
```

네이티브 API는 콜백 기반이라 복잡하다. 실무에서는 `idb` 라이브러리로 Promise 기반으로 감싸서 쓰는 경우가 많다.

```js
import { openDB } from "idb";

const db = await openDB("myDB", 1, {
  upgrade(db) {
    db.createObjectStore("todos", { keyPath: "id", autoIncrement: true });
  },
});

await db.add("todos", { title: "idb 라이브러리 사용", status: "todo" });
const todo = await db.get("todos", 1);
console.log(todo);
```

### 주요 활용 사례

- PWA의 오프라인 캐싱 (파일, 이미지, API 응답)
- 대규모 사용자 생성 데이터 (초안 자동 저장, 로컬 메모장)
- 복잡한 필터링·정렬이 필요한 클라이언트 데이터

---

## 네 가지 저장소 비교 정리

| 항목 | Cookie | localStorage | sessionStorage | IndexedDB |
| --- | --- | --- | --- | --- |
| 용량 | ~4KB | ~5~10MB | ~5~10MB | 수십~수백MB 이상 |
| 수명 | 설정한 만료 일자 | 명시적 삭제 전까지 | 탭 닫을 때까지 | 명시적 삭제 전까지 |
| 탭 공유 | 같은 오리진 공유 | 같은 오리진 공유 | 탭 단위 격리 | 같은 오리진 공유 |
| 서버 전송 | 자동으로 매 요청마다 | ❌ | ❌ | ❌ |
| API 방식 | 동기 (문자열) | 동기 (문자열) | 동기 (문자열) | 비동기 (객체) |
| 보안 속성 | HttpOnly, Secure | 없음 | 없음 | 없음 |
| 저장 형태 | 문자열 | 문자열 | 문자열 | JS 객체 (구조화) |

---

## 실무 선택 가이드

- **인증 토큰**: 서버에서 `HttpOnly + Secure + SameSite` 쿠키로 설정. XSS가 있어도 JS에서 접근 불가.
- **UI 설정 (테마, 언어)**: `localStorage`. 간단하고 영구적.
- **다단계 폼, 임시 상태**: `sessionStorage`. 탭을 닫으면 자연스럽게 정리됨.
- **대용량 오프라인 데이터, PWA 캐시**: `IndexedDB`. 비동기라 UX도 좋음.

---

## Storage 이벤트

`localStorage`는 **같은 오리진의 다른 탭에서 값이 변경될 때** `storage` 이벤트를 발생시킨다. 이를 활용해 여러 탭 간 상태를 동기화할 수 있다.

```js
window.addEventListener("storage", (event) => {
  if (event.key === "theme") {
    applyTheme(event.newValue);
  }
});
```

단, **같은 탭에서 변경한 경우에는 이 이벤트가 발생하지 않는다**는 점을 기억해야 한다.
