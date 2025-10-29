---
layout: post
title: "[Daily morning study] React의 useState와 useEffect 깊이 알기"
description: >
  #daily morning study
category: 
    - devlog
    - study
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---
---
title: "React의 useState와 useEffect 깊이 알기 — 학습 가이드"
---

이 문서는 useState와 useEffect를 실제로 공부하면서 정리한 내용이다. 핵심 개념, 동작 원리, 자주 겪는 함정과 해결법, 실전 코드 예시를 담았다. 복습용으로 바로 읽고 실습해볼 수 있게 작성했다.

목차
- useState: 기본과 내부 동작
- useState 고급 패턴과 성능 팁
- useEffect: 기본 동작과 생명주기
- 의존성 배열과 스테일 클로저 문제
- 정리된 예제들
- 실전 체크리스트와 자주 발생하는 버그/해결법
- 참고용 간단 패턴(커스텀 훅들)

---

useState: 기본과 내부 동작
- 선언
  - const [state, setState] = useState(initial)
  - initial은 값 또는 초기화 함수(() => expensiveInit())로 전달 가능. 함수형 초기화는 초기 렌더 때만 실행된다.
- setState의 동작
  - 값이 변경되면 컴포넌트는 리렌더된다. React는 내부적으로 Object.is(구조적 비교가 아닌 동일성 비교)를 사용해 변경 여부를 판정한다.
  - setState에 직접 값을 넣어도 되고(예: setCount(3)), 이전 값을 이용하는 함수형 업데이트(setCount(prev => prev + 1))도 가능하다.
- 함수형 업데이트의 장점
  - 비동기적 업데이트 환경(동시성, 여러 setState 호출)에서 안정적으로 이전 상태를 기반으로 계산할 때 필요하다.
  - 예: setCount(c => c + 1)

예시 — 지연 초기화
```jsx
import React, { useState } from 'react';

function expensiveInit() {
  // 비용이 큰 연산 예시
  let total = 0;
  for (let i = 0; i < 1e7; i++) total += i;
  return total;
}

function MyComponent() {
  const [value] = useState(() => expensiveInit()); // 초기화 함수로 한 번만 실행
  return <div>{value}</div>;
}
```

useState 고급 패턴과 성능 팁
- 여러 state 분리 vs 한 객체로 관리
  - 여러 개의 독립된 state로 분리하면 관련 없는 변경이 서로에게 영향을 덜 준다.
  - 복잡한 상태 전환 로직이나 연속적인 상태 변경은 useReducer가 더 명확할 수 있다.
- 여러 setState 호출의 배칭
  - 이벤트 핸들러 내부에서는 React가 여러 setState 호출을 한 번에 배치해 하나의 리렌더로 만든다.
  - React 18부터는 자동 배칭 범위가 넓어져 비동기 콜백(프로미스, setTimeout 등)에서도 배칭이 동작한다.
- 상태가 객체일 때 불변성 유지
  - setState(prev => ({ ...prev, updatedField: newValue })) 형식을 사용해 변경된 부분만 복사해줘야 한다.

useEffect: 기본 동작과 생명주기
- 선언
  - useEffect(() => { /* effect */ }, [deps])
  - deps가 비어있으면 마운트 시 한 번, deps가 없으면(정확히 deps 생략) 매 렌더마다 실행.
- 정리(cleanup)
  - effect 내부에서 반환한 함수는 cleanup으로 동작. 다음 실행 전에(또는 언마운트 시) 호출된다.
- 실행 타이밍
  - useEffect는 브라우저 페인트 이후 비동기로 실행된다(렌더 후 실행). UI가 먼저 그려진 다음 효과가 실행되므로 레이아웃 차단이 아니다.
  - useLayoutEffect는 paint 이전(동기), 레이아웃을 변경해야 할 때 사용.
- 정리 순서
  - 같은 효과 훅에서 deps가 변할 때 이전 effect의 cleanup이 실행된 뒤 새 effect가 실행된다.
  - 언마운트 시 모든 cleanup이 실행된다.

의존성 배열과 스테일 클로저(캐싱된 값) 문제
- 의존성 배열은 값의 참조 자체(Object.is 기준)를 비교한다. 배열/객체/함수는 매 렌더마다 새로운 참조이면 deps로 넣으면 effect는 계속 재실행된다.
- 스테일 클로저 문제(가장 흔한 실수)
  - effect 내부에서 사용한 state나 props를 deps에 넣지 않으면 effect 내부에서 오래된(stale) 값을 참조할 수 있다.
  - 해결 방법:
    - 필요한 값은 deps 배열에 명시한다.
    - 상태의 최신 값이 필요하고 deps로 넣는 것이 불가능하거나 불필요한 재실행을 유발하면 useRef로 최신 값을 저장하거나 setState의 함수형 업데이트를 사용한다.
    - 함수는 useCallback으로 감싸서 stable reference를 만들거나 deps에 포함시킨다.

스테일 클로저 예시 — 잘못된 setInterval 사용
```jsx
import React, { useState, useEffect } from 'react';

function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      // 이 콜백은 처음 렌더 시점의 count를 캡처한다 -> 값이 증가하지 않음
      setCount(count + 1);
    }, 1000);
    return () => clearInterval(id);
  }, []); // 빈 deps -> 콜백 안의 count는 고정된 값

  return <div>{count}</div>;
}
```

해결 1 — 함수형 업데이트 사용
```jsx
useEffect(() => {
  const id = setInterval(() => {
    setCount(c => c + 1); // 항상 최신 상태 기반으로 업데이트
  }, 1000);
  return () => clearInterval(id);
}, []);
```

해결 2 — ref로 최신 값 유지(외부에서 읽어야 할 때)
```jsx
import React, { useRef, useEffect, useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);
  const countRef = useRef(count);
  countRef.current = count; // 항상 최신 값 유지

  useEffect(() => {
    const id = setInterval(() => {
      // 외부 API 호출 등에서 최신값을 읽어야 할 때 사용
      console.log('latest count:', countRef.current);
      setCount(c => c + 1);
    }, 1000);
    return () => clearInterval(id);
  }, []);

  return <div>{count}</div>;
}
```

비동기 작업과 취소
- effect 함수는 async로 만들면 안 된다(반환값이 Promise가 되어 cleanup 역할을 하지 못함).
- fetch 등 비동기 작업에는 내부 async 함수를 만들고 AbortController로 취소하거나, 플래그(ref)를 사용해 마운트 여부를 체크한다.
- React의 Strict Mode와 개발환경에서는 effect가 두 번 호출될 수 있으므로 부수효과에 idempotent(중복 실행해도 안전) 특성이 있는지 확인한다.

fetch 예시(AbortController)
```jsx
import React, { useEffect, useState } from 'react';

function DataFetcher({ url }) {
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);

  useEffect(() => {
    const controller = new AbortController();
    const signal = controller.signal;

    async function fetchData() {
      try {
        const res = await fetch(url, { signal });
        if (!res.ok) throw new Error('네트워크 오류');
        const json = await res.json();
        setData(json);
      } catch (err) {
        if (err.name === 'AbortError') return; // 취소된 경우 무시
        setError(err);
      }
    }

    fetchData();

    return () => controller.abort(); // 언마운트 또는 url 변경 시 취소
  }, [url]);

  if (error) return <div>Error</div>;
  if (!data) return <div>Loading...</div>;
  return <pre>{JSON.stringify(data, null, 2)}</pre>;
}
```

useEffect vs useLayoutEffect 요약
- useLayoutEffect: 마운트/업데이트 직후, 브라우저가 페인트하기 전에 동기적으로 실행. 레이아웃 측정/동기 DOM 변형 필요 시 사용.
- useEffect: 비동기적으로 페인트 후 실행. 대부분의 사이드 이펙트는 이걸 사용.

Strict Mode, 개발환경과 효과의 중복 실행
- React Strict Mode(개발 모드)는 일부 사이드이펙트를 감지하기 위해 컴포넌트의 마운트/언마운트를 의도적으로 두 번 실행한다(특히 초기 마운트). 따라서 effect가 두 번 실행되어도 안전한지 확인해야 한다.
- 반드시 cleanup이 적절히 구현되어 있어야 메모리 누수/이중 API 호출 등을 막을 수 있다.

실전 예제 모음

1) 카운터와 배칭/함수형 업데이트
```jsx
function Counter() {
  const [count, setCount] = useState(0);

  function incrementTwice() {
    // 함수형 업데이트를 사용하면 안전하게 누적 업데이트 가능
    setCount(c => c + 1);
    setCount(c => c + 1);
  }

  return (
    <div>
      <div>{count}</div>
      <button onClick={incrementTwice}>+2</button>
    </div>
  );
}
```

2) 의존성 배열이 복잡할 때의 패턴 — debounce 훅
```jsx
import { useEffect, useState } from 'react';

function useDebouncedValue(value, delay = 300) {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id); // value가 바뀌면 이전 타이머 취소
  }, [value, delay]);

  return debounced;
}
```

3) 이전 값 추적(usePrevious)
```jsx
import { useRef, useEffect } from 'react';

function usePrevious(value) {
  const ref = useRef();
  useEffect(() => {
    ref.current = value;
  }, [value]);
  return ref.current;
}
```

실전 체크리스트 (useEffect 작성 시)
- effect 내부에서 사용하는 모든 외부 값(state, props, 함수)은 deps에 넣었는가?
- deps에 넣어야 하는데 넣으면 불필요한 재실행을 유발하는 값(예: 객체/함수)은 useMemo/useCallback으로 안정화했는가?
- cleanup 함수가 적절히 정리하는가?(타이머, 구독, 이벤트 리스너, abort controller)
- effect가 idempotent한가? (동일한 effect가 여러 번 실행되어도 문제 없는가)
- 비동기 로직에서 setState를 호출하기 전에 컴포넌트가 언마운트 되었는지 확인하는가?(AbortController 또는 mounted ref 사용)
- 렌더 타이밍(레이아웃 측정이나 DOM 변경 필요 시 useLayoutEffect 사용 고려)

자주 발생하는 버그와 해결법
- 버그: setState 후 최신 상태가 반영되지 않음
  - 원인: setState에 기존 상태를 사용해야 하는데 직접 값을 사용했거나 클로저로 인해 옛값 참조
  - 해결: 함수형 업데이트 setState(prev => newVal(prev)) 사용
- 버그: effect가 의도치 않게 매우 자주 실행됨
  - 원인: deps 배열에 매번 다른 참조(객체/함수)를 넣음
  - 해결: deps로 넣을 필요 없는 값인지 검토, useCallback/useMemo로 안정화
- 버그: 언마운트 후 setState 경고
  - 원인: 비동기 작업이 완료되어 setState를 호출하는데 컴포넌트가 이미 언마운트됨
  - 해결: AbortController 사용 또는 isMounted ref 체크 후 setState 하지 않음
- 버그: Strict Mode에서 effect가 두 번 실행되어 API를 두 번 호출
  - 해결: effect가 두 번 실행되어도 안전한지 만들거나 cleanup/취소 로직을 추가

테스트 팁
- jest + react-testing-library에서 effect를 테스트할 때는 act/async waitFor 를 사용해 비동기 사이드 이펙트가 처리되도록 한다.
- 타이머 관련 로직(setTimeout 등)은 jest.useFakeTimers()로 제어해서 검사한다.

권장 패턴 정리
- 상태 변화 로직이 복잡하면 useReducer로 대체해 로직과 업데이트를 명확히 한다.
- DOM 변경/측정이 필요하면 useLayoutEffect, 대부분의 사이드 이펙트는 useEffect.
- 커스텀 훅으로 반복되는 상태/이펙트 로직 추출(예: fetch 훅, debounce 훅, interval 훅).
- eslint-plugin-react-hooks의 exhaustive-deps 규칙을 활성화해 deps 실수를 줄인다. (경우에 따라 내부 주석으로 예외 처리)

간단한 커스텀 훅 예: useMountedRef
```jsx
import { useEffect, useRef } from 'react';

function useMountedRef() {
  const mountedRef = useRef(false);
  useEffect(() => {
    mountedRef.current = true;
    return () => { mountedRef.current = false; };
  }, []);
  return mountedRef;
}

// 사용 예
function Component() {
  const mountedRef = useMountedRef();
  useEffect(() => {
    fetch('/api').then(res => res.json()).then(data => {
      if (!mountedRef.current) return;
      // 안전하게 setState
    });
  }, []);
}
```

마무리 요약
- useState: 값 또는 lazy initializer, 함수형 업데이트를 이해하자. 상태를 어떻게 분리할지 설계하면 렌더 최적화에 유리하다.
- useEffect: deps 배열, cleanup, 실행 타이밍(useLayoutEffect와 비교)을 확실히 하자. 스테일 클로저와 비동기 취소가 핵심 문제다.
- 실전에서는 함수형 업데이트, AbortController, useRef, useCallback/useMemo를 적절히 조합해 안정적이고 효율적인 상태/사이드 이펙트를 만들자.

참고(간단)
- React 문서: Hooks FAQ, Using the Effect Hook, Rules of Hooks
- eslint-plugin-react-hooks: exhaustive-deps 규칙

끝.
