---
layout: post
title: "[Daily morning study] React 훅(Hooks) 동작 원리와 규칙"
description: >
  #daily morning study
category: 
    - dms
    - dms-frontend
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## React 훅(Hooks)이란

React 16.8부터 도입된 기능으로, 함수형 컴포넌트에서도 상태 관리와 생명주기 기능을 사용할 수 있게 해준다. 훅 이전에는 이런 기능을 쓰려면 반드시 클래스 컴포넌트를 써야 했다.

훅이 나오면서 클래스 컴포넌트 없이도 상태, 사이드 이펙트, 컨텍스트 등을 함수형으로 처리할 수 있게 됐다.

---

## 훅의 내부 동작 원리

React는 훅의 상태를 **컴포넌트 인스턴스에 연결된 연결 리스트(linked list)** 형태로 관리한다.

컴포넌트가 처음 렌더링될 때 각 훅 호출이 순서대로 리스트에 등록된다. 이후 리렌더링 시에는 같은 순서로 리스트를 순회하면서 이전 상태를 꺼내온다.

```
// 내부적으로 이런 식으로 동작한다 (단순화된 구조)
let hooks = [];
let currentIndex = 0;

function useState(initialValue) {
  const index = currentIndex++;
  if (hooks[index] === undefined) {
    hooks[index] = initialValue; // 최초 등록
  }
  const setState = (newValue) => {
    hooks[index] = newValue;
    rerender(); // 리렌더 트리거
  };
  return [hooks[index], setState];
}
```

이 구조 때문에 훅 호출 순서가 매 렌더링마다 동일해야 한다. 조건문 안에서 훅을 호출하면 순서가 달라져서 엉뚱한 상태를 참조하게 된다.

---

## 훅의 두 가지 규칙

### 규칙 1: 최상위 레벨에서만 호출

```jsx
// Bad
function Component({ isLoggedIn }) {
  if (isLoggedIn) {
    const [name, setName] = useState(''); // ❌ 조건문 안에서 훅 호출
  }
}

// Good
function Component({ isLoggedIn }) {
  const [name, setName] = useState(''); // ✅ 최상위에서 호출
  if (!isLoggedIn) return null;
  // ...
}
```

반복문, 조건문, 중첩 함수 안에서 훅을 호출하면 렌더링마다 호출 횟수가 달라질 수 있다.

### 규칙 2: React 함수 컴포넌트 또는 커스텀 훅에서만 호출

일반 JavaScript 함수에서는 훅을 사용할 수 없다. React가 내부적으로 현재 렌더링 중인 컴포넌트를 추적하면서 훅 상태를 관리하기 때문이다.

---

## 주요 훅 정리

### useState

가장 기본적인 상태 관리 훅. 상태 값과 업데이트 함수를 반환한다.

```jsx
const [count, setCount] = useState(0);

// 이전 상태를 기반으로 업데이트할 때는 함수형 업데이트 권장
setCount(prev => prev + 1);
```

초기값으로 함수를 넘기면 최초 렌더링 시에만 실행된다 (지연 초기화).

```jsx
const [data, setData] = useState(() => expensiveComputation()); // lazy initialization
```

### useEffect

사이드 이펙트를 처리하는 훅. 렌더링 이후에 실행된다.

```jsx
useEffect(() => {
  // 실행 코드 (API 호출, 구독, 타이머 등)
  const subscription = subscribe(id);

  return () => {
    subscription.unsubscribe(); // 클린업 함수
  };
}, [id]); // 의존성 배열
```

의존성 배열 동작:

| 의존성 배열 | 실행 시점 |
|-------------|-----------|
| 없음 | 모든 렌더링 후 |
| `[]` | 마운트 시 한 번만 |
| `[a, b]` | a 또는 b가 변경될 때 |

클린업 함수는 다음 이펙트 실행 전과 언마운트 시 호출된다.

### useRef

렌더링을 트리거하지 않고 값을 유지하거나, DOM 노드를 참조할 때 쓴다.

```jsx
// DOM 참조
const inputRef = useRef(null);
<input ref={inputRef} />
inputRef.current.focus();

// 이전 값 추적 (렌더링 유발 없음)
const prevCount = useRef(count);
useEffect(() => {
  prevCount.current = count;
});
```

`useRef`가 반환하는 객체는 컴포넌트 생애주기 동안 동일한 참조를 유지한다.

### useMemo / useCallback

불필요한 연산이나 함수 재생성을 방지하는 최적화 훅.

```jsx
// useMemo: 값 메모이제이션
const expensiveValue = useMemo(() => {
  return heavyComputation(a, b);
}, [a, b]);

// useCallback: 함수 메모이제이션
const handleClick = useCallback(() => {
  doSomething(id);
}, [id]);
```

의존성 배열의 값이 바뀌지 않으면 이전에 계산된 값/함수를 재사용한다.

주의: 모든 곳에 남발하면 오히려 메모이제이션 비교 비용이 생긴다. 실제로 무거운 연산이나 자식 컴포넌트에 props로 전달하는 함수에 적용하는 게 맞다.

### useContext

Context API의 값을 구독하는 훅. Provider 아래에서 value를 꺼내 쓸 수 있다.

```jsx
const ThemeContext = createContext('light');

function App() {
  return (
    <ThemeContext.Provider value="dark">
      <Child />
    </ThemeContext.Provider>
  );
}

function Child() {
  const theme = useContext(ThemeContext); // 'dark'
  return <div className={theme}>...</div>;
}
```

Context 값이 바뀌면 해당 Context를 구독하는 컴포넌트가 모두 리렌더링된다.

### useReducer

복잡한 상태 로직을 reducer 패턴으로 관리할 때 쓴다.

```jsx
function reducer(state, action) {
  switch (action.type) {
    case 'increment': return { count: state.count + 1 };
    case 'decrement': return { count: state.count - 1 };
    default: return state;
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, { count: 0 });
  return (
    <>
      <span>{state.count}</span>
      <button onClick={() => dispatch({ type: 'increment' })}>+</button>
    </>
  );
}
```

상태 전환 로직이 여러 곳에서 쓰이거나 다음 상태가 이전 상태에 복잡하게 의존할 때 `useState`보다 적합하다.

---

## 커스텀 훅(Custom Hook)

반복되는 훅 로직을 추출해서 재사용 가능한 함수로 만든 것. 이름이 반드시 `use`로 시작해야 훅 규칙 린터가 적용된다.

```jsx
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    setLoading(true);
    fetch(url)
      .then(res => res.json())
      .then(setData)
      .catch(setError)
      .finally(() => setLoading(false));
  }, [url]);

  return { data, loading, error };
}

// 사용
function UserProfile({ userId }) {
  const { data, loading } = useFetch(`/api/users/${userId}`);
  if (loading) return <Spinner />;
  return <div>{data.name}</div>;
}
```

커스텀 훅은 상태 자체를 공유하지 않는다. 훅을 호출하는 각 컴포넌트가 독립적인 상태를 가진다.

---

## 훅 관련 자주 나오는 문제들

### 클로저 스탈(Stale Closure)

이펙트나 콜백에서 최신 상태가 아닌 이전 렌더링의 값을 참조하는 문제.

```jsx
function Timer() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1); // ❌ 클로저가 최초 count=0을 캡처
    }, 1000);
    return () => clearInterval(id);
  }, []); // 의존성 배열 비어있음
}
```

해결 방법: 함수형 업데이트 사용 또는 의존성 배열에 추가.

```jsx
setCount(prev => prev + 1); // ✅ 이전 상태를 기반으로 업데이트
```

### 무한 루프

`useEffect` 내에서 의존성 배열에 있는 상태를 업데이트하면 무한 루프 발생.

```jsx
// ❌ 무한 루프
useEffect(() => {
  setData(processData(data));
}, [data]);

// ✅ 올바른 처리
useEffect(() => {
  setData(processData(rawData));
}, [rawData]); // rawData는 외부에서 오는 값
```

---

## 클래스 컴포넌트 생명주기와 비교

| 클래스 컴포넌트 | 훅 |
|----------------|----|
| `componentDidMount` | `useEffect(() => {}, [])` |
| `componentDidUpdate` | `useEffect(() => {}, [dep])` |
| `componentWillUnmount` | `useEffect(() => { return cleanup; }, [])` |
| `this.setState` | `useState` |
| `shouldComponentUpdate` | `React.memo` + `useMemo` |

훅은 생명주기를 시점 기준이 아닌 동기화 기준으로 생각하게 해준다. "마운트 시에 뭘 해야 한다"가 아니라 "이 값이 바뀌면 이걸 동기화해야 한다"는 관점이다.
