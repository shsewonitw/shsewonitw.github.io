---
layout: post
title: "[Daily morning study] React 상태 관리 라이브러리 비교 - Redux, Zustand, Jotai"
description: >
  #daily morning study
category: 
    - dms
    - -front
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 왜 전역 상태 관리가 필요한가

React 컴포넌트는 `props`와 `state`로 데이터를 다루는데, 컴포넌트 트리가 깊어질수록 상위에서 하위로 데이터를 전달하기 위해 중간 컴포넌트들이 관계없는 props를 계속 전달받아야 하는 **props drilling** 문제가 생긴다.

이를 해결하기 위해 전역 상태 관리 라이브러리를 사용한다. 현재 가장 많이 쓰이는 세 가지를 비교해본다.

---

## Context API와 useReducer

라이브러리 전에, React 내장 기능으로도 전역 상태를 관리할 수 있다.

```jsx
const CountContext = React.createContext(null);

function CountProvider({ children }) {
  const [count, setCount] = React.useState(0);
  return (
    <CountContext.Provider value={{ count, setCount }}>
      {children}
    </CountContext.Provider>
  );
}

function Counter() {
  const { count, setCount } = React.useContext(CountContext);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

**문제점**: Context 값이 바뀌면 해당 Context를 구독하는 모든 컴포넌트가 리렌더링된다. 상태가 많아질수록 불필요한 리렌더링이 급증해 성능 문제가 생긴다.

---

## Redux

### 핵심 개념

Redux는 **단방향 데이터 흐름**과 **단일 스토어(Single Source of Truth)** 원칙을 따른다.

```
Action → Reducer → Store → View → Action (순환)
```

- **Store**: 애플리케이션 전체 상태를 담는 하나의 객체
- **Action**: 상태 변경 의도를 나타내는 순수 객체 `{ type, payload }`
- **Reducer**: `(state, action) => newState` 형태의 순수 함수
- **Dispatch**: Action을 Store에 보내는 함수

### Redux Toolkit (RTK) — 현대 Redux

예전 Redux는 보일러플레이트 코드가 너무 많았다. RTK로 대폭 줄어들었다.

```js
import { createSlice, configureStore } from '@reduxjs/toolkit';

const counterSlice = createSlice({
  name: 'counter',
  initialState: { value: 0 },
  reducers: {
    increment: (state) => { state.value += 1; },      // Immer로 불변성 자동 처리
    decrement: (state) => { state.value -= 1; },
    incrementByAmount: (state, action) => {
      state.value += action.payload;
    },
  },
});

export const { increment, decrement, incrementByAmount } = counterSlice.actions;

const store = configureStore({
  reducer: { counter: counterSlice.reducer },
});
```

```jsx
// 컴포넌트에서 사용
import { useSelector, useDispatch } from 'react-redux';

function Counter() {
  const count = useSelector((state) => state.counter.value);
  const dispatch = useDispatch();

  return (
    <div>
      <button onClick={() => dispatch(increment())}>{count}</button>
    </div>
  );
}
```

### 비동기 처리: createAsyncThunk

```js
const fetchUser = createAsyncThunk('user/fetch', async (userId) => {
  const res = await fetch(`/api/users/${userId}`);
  return res.json();
});

const userSlice = createSlice({
  name: 'user',
  initialState: { data: null, loading: false, error: null },
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(fetchUser.pending, (state) => { state.loading = true; })
      .addCase(fetchUser.fulfilled, (state, action) => {
        state.loading = false;
        state.data = action.payload;
      })
      .addCase(fetchUser.rejected, (state, action) => {
        state.loading = false;
        state.error = action.error.message;
      });
  },
});
```

**Redux가 적합한 상황**: 대규모 팀, 복잡한 상태 로직, Redux DevTools로 시간 여행 디버깅이 필요한 경우

---

## Zustand

Zustand는 독일어로 "상태"를 뜻한다. 매우 가볍고 보일러플레이트가 거의 없다.

### 기본 사용법

```js
import { create } from 'zustand';

const useCounterStore = create((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
  decrement: () => set((state) => ({ count: state.count - 1 })),
  reset: () => set({ count: 0 }),
}));

// 컴포넌트에서
function Counter() {
  const { count, increment } = useCounterStore();
  return <button onClick={increment}>{count}</button>;
}
```

### 선택적 구독으로 리렌더링 최적화

```js
// count만 구독 → count가 바뀔 때만 리렌더링
const count = useCounterStore((state) => state.count);

// 여러 값 구독: shallow 비교
import { shallow } from 'zustand/shallow';
const { count, name } = useCounterStore(
  (state) => ({ count: state.count, name: state.name }),
  shallow
);
```

### 비동기 처리

```js
const useUserStore = create((set) => ({
  user: null,
  loading: false,
  fetchUser: async (id) => {
    set({ loading: true });
    const res = await fetch(`/api/users/${id}`);
    const data = await res.json();
    set({ user: data, loading: false });
  },
}));
```

Zustand는 별도의 Provider 없이 동작한다는 점이 큰 장점이다. 스토어 자체가 모듈 스코프에 위치하기 때문에 Provider 래핑이 불필요하다.

---

## Jotai

Jotai는 React의 `useState`를 전역으로 확장한 것에 가깝다. **Atom** 단위로 상태를 쪼개는 **원자적(Atomic) 상태 관리** 방식이다.

### 기본 사용법

```js
import { atom, useAtom, useAtomValue, useSetAtom } from 'jotai';

const countAtom = atom(0);

function Counter() {
  const [count, setCount] = useAtom(countAtom);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// 읽기만 할 때
function Display() {
  const count = useAtomValue(countAtom);
  return <span>{count}</span>;
}

// 쓰기만 할 때
function ResetButton() {
  const setCount = useSetAtom(countAtom);
  return <button onClick={() => setCount(0)}>Reset</button>;
}
```

### 파생 Atom (derived atom)

```js
const priceAtom = atom(100);
const quantityAtom = atom(3);

// 읽기 전용 파생 atom
const totalAtom = atom((get) => get(priceAtom) * get(quantityAtom));

// 읽기/쓰기 atom
const doubledCountAtom = atom(
  (get) => get(countAtom) * 2,
  (get, set, newValue) => set(countAtom, newValue / 2)
);
```

### 비동기 Atom

```js
const userAtom = atom(async (get) => {
  const id = get(userIdAtom);
  const res = await fetch(`/api/users/${id}`);
  return res.json();
});

// Suspense와 자연스럽게 통합
function UserProfile() {
  const user = useAtomValue(userAtom); // 자동으로 Suspense 처리
  return <div>{user.name}</div>;
}
```

Jotai는 사용하는 atom만 리렌더링을 유발하기 때문에 Context보다 훨씬 세밀한 리렌더링 제어가 가능하다.

---

## 세 라이브러리 비교

| 항목 | Redux (RTK) | Zustand | Jotai |
|------|-------------|---------|-------|
| 번들 크기 | ~47KB | ~3KB | ~8KB |
| 보일러플레이트 | 중간 (RTK로 감소) | 매우 적음 | 거의 없음 |
| 스토어 구조 | 단일 중앙 스토어 | 여러 스토어 가능 | atom 단위 분산 |
| 리렌더링 최적화 | selector 사용 | selector 사용 | atom 단위 자동 |
| DevTools | 강력한 Redux DevTools | 지원 (미들웨어) | 지원 (jotai-devtools) |
| 비동기 처리 | createAsyncThunk | 스토어 내 직접 | async atom |
| Provider 필요 | 필요 (Provider) | 불필요 | 필요 (Provider) |
| 러닝 커브 | 높음 | 낮음 | 낮음~중간 |
| 적합한 규모 | 대규모 프로젝트 | 중소규모 | 중소~대규모 |

---

## 어떤 걸 선택해야 하나

### Redux (RTK)를 쓸 때

- 팀 규모가 크고 상태 변경 이력 추적이 중요한 경우
- 복잡한 미들웨어 로직이 필요한 경우
- 이미 Redux를 사용 중인 레거시 프로젝트 유지보수

### Zustand를 쓸 때

- 빠르게 전역 상태가 필요한 중소규모 프로젝트
- Redux의 보일러플레이트 없이 간단하게 사용하고 싶은 경우
- 여러 독립적인 스토어를 분리해서 관리하고 싶은 경우

### Jotai를 쓸 때

- 상태를 잘게 쪼개어 관리하고 싶은 경우
- React Suspense와 통합하여 비동기 처리를 깔끔하게 하고 싶은 경우
- 컴포넌트 단위로 세밀한 리렌더링 최적화가 필요한 경우

---

## 리렌더링 최적화 관점에서의 차이

Context API의 문제를 생각해보면 이해가 쉽다.

```
Context 값 변경 → 구독하는 모든 컴포넌트 리렌더링  (Context API)
스토어 값 변경 → selector가 반환하는 값이 변경된 컴포넌트만 리렌더링  (Redux, Zustand)
atom 값 변경 → 해당 atom을 구독하는 컴포넌트만 리렌더링  (Jotai)
```

특히 Jotai의 atom 방식은 React의 `useState`와 동일한 방식으로 렌더링 범위를 결정하기 때문에, 개발자가 명시적으로 selector를 작성하지 않아도 자동으로 최적화된다는 장점이 있다.

---

## 정리

상태 관리 라이브러리를 선택할 때 "무조건 최신 것이 좋다"는 생각보다 팀 규모, 프로젝트 복잡도, 팀원의 학습 비용을 함께 고려해야 한다. 요즘 중소규모 프로젝트에서는 Zustand가 가장 자주 선택되는 편이고, 서버 상태(비동기 데이터 fetching, 캐싱)는 React Query(TanStack Query)와 조합하는 패턴이 사실상 표준이 되고 있다.
