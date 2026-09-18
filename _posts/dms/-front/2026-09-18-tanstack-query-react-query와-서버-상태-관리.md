---
layout: post
title: "[Daily morning study] TanStack Query(React Query)와 서버 상태 관리"
description: >
  #daily morning study
category: 
    - dms
    - -front
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 서버 상태 vs 클라이언트 상태

프론트엔드 상태(State)는 크게 두 종류로 나눌 수 있다.

| 구분 | 예시 | 특징 |
|------|------|------|
| 클라이언트 상태 | 모달 열림 여부, 선택된 탭, 입력 중인 폼 값 | 앱 내에서 완전히 제어 가능 |
| 서버 상태 | API 응답 데이터, 사용자 목록, 게시글 등 | 서버와 동기화 필요, 백그라운드에서 변경될 수 있음 |

Redux나 Zustand 같은 클라이언트 상태 관리 라이브러리를 서버 데이터 캐싱에도 쓰다 보면 금방 복잡해진다. 로딩 상태, 에러 처리, 캐시 갱신, 중복 요청 방지 같은 로직을 직접 구현해야 하기 때문이다.

TanStack Query(구 React Query)는 이 서버 상태 문제를 전담하는 라이브러리다.

---

## TanStack Query 핵심 개념

### QueryClient와 QueryClientProvider

앱 최상단에 `QueryClientProvider`를 감싸서 전역 캐시를 공유한다.

```tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

const queryClient = new QueryClient();

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <MyApp />
    </QueryClientProvider>
  );
}
```

`QueryClient`는 내부적으로 캐시 저장소 역할을 한다. 동일한 query key를 쓰는 컴포넌트들은 같은 캐시 엔트리를 공유한다.

---

### useQuery — 데이터 조회

```tsx
import { useQuery } from '@tanstack/react-query';

function UserList() {
  const { data, isPending, isError, error } = useQuery({
    queryKey: ['users'],
    queryFn: () => fetch('/api/users').then(res => res.json()),
  });

  if (isPending) return <div>Loading...</div>;
  if (isError) return <div>Error: {error.message}</div>;

  return (
    <ul>
      {data.map(user => <li key={user.id}>{user.name}</li>)}
    </ul>
  );
}
```

- `queryKey`: 캐시 키 역할. `['users', userId]`처럼 배열로 계층화할 수 있다.
- `queryFn`: 실제 데이터를 가져오는 비동기 함수.
- `isPending`: 최초 로딩 중인 상태 (캐시 없음 + 요청 중).
- `isFetching`: 백그라운드 refetch 포함한 모든 요청 중 상태.

---

### useMutation — 데이터 변경

```tsx
import { useMutation, useQueryClient } from '@tanstack/react-query';

function CreateUserForm() {
  const queryClient = useQueryClient();

  const mutation = useMutation({
    mutationFn: (newUser) =>
      fetch('/api/users', {
        method: 'POST',
        body: JSON.stringify(newUser),
      }).then(res => res.json()),
    onSuccess: () => {
      // 'users' 캐시를 무효화 → 자동 refetch 트리거
      queryClient.invalidateQueries({ queryKey: ['users'] });
    },
  });

  return (
    <button onClick={() => mutation.mutate({ name: 'Alice' })}>
      {mutation.isPending ? '저장 중...' : '사용자 추가'}
    </button>
  );
}
```

- `mutationFn`: POST/PUT/DELETE 같은 쓰기 요청.
- `onSuccess/onError/onSettled`: 결과에 따른 사이드 이펙트 처리.
- `invalidateQueries`: 해당 키 캐시를 stale 처리해서 다음 접근 시 새로 fetch하게 만든다.

---

## 캐싱 전략: staleTime과 gcTime

TanStack Query는 데이터를 두 단계로 관리한다.

```
fresh → stale → inactive → garbage collected
```

| 옵션 | 기본값 | 의미 |
|------|--------|------|
| `staleTime` | 0ms | fresh 상태 유지 시간. 이 시간 내엔 refetch 안 함 |
| `gcTime` (구 `cacheTime`) | 5분 | inactive 상태 캐시를 메모리에 유지하는 시간 |

```tsx
const { data } = useQuery({
  queryKey: ['config'],
  queryFn: fetchConfig,
  staleTime: 1000 * 60 * 5,  // 5분간 fresh → 불필요한 요청 차단
  gcTime: 1000 * 60 * 10,    // 10분간 캐시 유지
});
```

- `staleTime: Infinity`: 한 번 받아온 데이터를 절대 재요청하지 않음 (정적 데이터에 적합).
- `staleTime: 0` (기본값): 화면에 돌아올 때마다 백그라운드에서 최신 데이터 확인.

---

## 자동 refetch 트리거

기본적으로 TanStack Query는 다음 상황에서 자동으로 최신 데이터를 가져온다.

| 트리거 | 기본값 | 설명 |
|--------|--------|------|
| `refetchOnWindowFocus` | `true` | 탭/창 포커스 복귀 시 |
| `refetchOnMount` | `true` | 컴포넌트 마운트 시 |
| `refetchOnReconnect` | `true` | 네트워크 재연결 시 |
| `refetchInterval` | `false` | 폴링 간격 (ex: 30초마다 자동 갱신) |

```tsx
const { data } = useQuery({
  queryKey: ['notifications'],
  queryFn: fetchNotifications,
  refetchInterval: 30_000,       // 30초마다 자동 갱신
  refetchOnWindowFocus: true,    // 탭 돌아올 때마다 갱신
});
```

---

## Query Key 설계

쿼리 키는 캐시 식별자이자 의존성 선언이다. 키가 바뀌면 새로운 요청이 나간다.

```tsx
// 목록 vs 단건
useQuery({ queryKey: ['users'] });
useQuery({ queryKey: ['users', userId] });

// 필터/페이지 포함
useQuery({ queryKey: ['users', { status: 'active', page: 2 }] });

// 파라미터가 바뀌면 자동으로 새 요청
const [page, setPage] = useState(1);
useQuery({
  queryKey: ['products', page],
  queryFn: () => fetchProducts(page),
});
```

`invalidateQueries({ queryKey: ['users'] })`는 `['users']`로 시작하는 모든 캐시(예: `['users', 1]`, `['users', 2]`)를 한 번에 무효화한다.

---

## Optimistic Update

서버 응답을 기다리지 않고 UI를 먼저 업데이트한 뒤, 실패하면 롤백하는 패턴이다.

```tsx
const mutation = useMutation({
  mutationFn: updateTodo,
  onMutate: async (newTodo) => {
    // 진행 중인 refetch 취소
    await queryClient.cancelQueries({ queryKey: ['todos', newTodo.id] });

    // 현재 값 스냅샷
    const previous = queryClient.getQueryData(['todos', newTodo.id]);

    // 낙관적으로 캐시 업데이트
    queryClient.setQueryData(['todos', newTodo.id], newTodo);

    return { previous };
  },
  onError: (err, newTodo, context) => {
    // 실패 시 스냅샷으로 롤백
    queryClient.setQueryData(['todos', newTodo.id], context.previous);
  },
  onSettled: (data, err, newTodo) => {
    // 성공/실패 무관하게 서버 데이터로 동기화
    queryClient.invalidateQueries({ queryKey: ['todos', newTodo.id] });
  },
});
```

---

## Dependent Query (의존 쿼리)

```tsx
const { data: user } = useQuery({
  queryKey: ['user', userId],
  queryFn: () => fetchUser(userId),
});

// user 데이터가 있어야 실행
const { data: posts } = useQuery({
  queryKey: ['posts', user?.id],
  queryFn: () => fetchPosts(user.id),
  enabled: !!user,  // user가 undefined면 쿼리 실행 안 함
});
```

`enabled` 옵션으로 조건부 쿼리를 만들 수 있다.

---

## Redux/Zustand와 비교

| 관심사 | TanStack Query | Redux / Zustand |
|--------|----------------|-----------------|
| 서버 데이터 캐싱 | ✅ 핵심 기능 | ❌ 직접 구현 필요 |
| 로딩/에러 상태 | ✅ 자동 제공 | ❌ 수동 관리 |
| 백그라운드 동기화 | ✅ 자동 | ❌ 직접 구현 필요 |
| UI 상태 관리 | ❌ 부적합 | ✅ 핵심 용도 |
| 복잡한 클라이언트 로직 | ❌ 부적합 | ✅ 적합 |

실무에서는 **TanStack Query로 서버 상태를 처리하고, Zustand나 Context API로 클라이언트 상태를 관리**하는 조합이 일반적이다. Redux를 굳이 붙일 이유가 크게 줄어든다.

---

## 요약

- **TanStack Query**는 서버 데이터의 fetch, 캐싱, 동기화, 갱신을 전담하는 라이브러리다.
- `staleTime`으로 불필요한 네트워크 요청을 줄이고, `gcTime`으로 메모리 사용을 제어한다.
- `queryKey`가 캐시의 식별자이자 의존성 배열 역할을 한다.
- Mutation 이후 `invalidateQueries`로 관련 캐시를 무효화해서 데이터 일관성을 유지한다.
- 서버 상태와 클라이언트 상태를 분리하면 코드가 훨씬 단순해진다.
