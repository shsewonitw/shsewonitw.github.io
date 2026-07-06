---
layout: post
title: "[Daily morning study] 유니온 파인드 (Union-Find / Disjoint Set)"
description: >
  #daily morning study
category: 
    - dms
    - dms-dsa
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 유니온 파인드란?

유니온 파인드(Union-Find)는 **서로소 집합(Disjoint Set)**을 효율적으로 관리하는 자료구조다. 여러 원소들을 그룹(집합)으로 묶고, 두 원소가 같은 집합에 속하는지 빠르게 판별하는 용도로 쓰인다.

핵심 연산은 두 가지다.

- **Find**: 특정 원소가 어느 집합(루트 노드)에 속하는지 반환
- **Union**: 두 집합을 하나로 합침

그래프에서 사이클 감지, 크루스칼 MST, 네트워크 연결 여부 확인 등에 자주 등장한다.

---

## 기본 구현

각 원소마다 `parent` 배열을 두고, 자기 자신을 부모로 초기화하는 것에서 시작한다.

```python
class UnionFind:
    def __init__(self, n):
        self.parent = list(range(n))  # 초기엔 자기 자신이 루트

    def find(self, x):
        if self.parent[x] != x:
            return self.find(self.parent[x])  # 루트까지 재귀 탐색
        return x

    def union(self, x, y):
        rx, ry = self.find(x), self.find(y)
        if rx != ry:
            self.parent[rx] = ry  # rx의 루트를 ry로 연결
```

위 구현은 동작은 하지만, 최악의 경우(일직선 트리) Find의 시간복잡도가 O(N)이 된다.

---

## 경로 압축 (Path Compression)

Find를 호출할 때, 거쳐간 모든 노드를 루트에 직접 연결해 버리는 최적화 기법이다.

```python
def find(self, x):
    if self.parent[x] != x:
        self.parent[x] = self.find(self.parent[x])  # 루트를 직접 가리키도록
    return self.parent[x]
```

한 번 Find를 실행하고 나면 중간 노드들이 전부 루트에 달라붙어, 이후 탐색이 O(1)에 가까워진다.

---

## 랭크 기반 Union (Union by Rank)

단순히 `parent[rx] = ry`로 합치면 트리가 한쪽으로 치우칠 수 있다. **랭크(트리 높이의 상한)**가 낮은 트리를 높은 트리 아래에 붙이면 균형을 유지할 수 있다.

```python
class UnionFind:
    def __init__(self, n):
        self.parent = list(range(n))
        self.rank = [0] * n

    def find(self, x):
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])
        return self.parent[x]

    def union(self, x, y):
        rx, ry = self.find(x), self.find(y)
        if rx == ry:
            return
        if self.rank[rx] < self.rank[ry]:
            rx, ry = ry, rx       # rx가 항상 rank가 크거나 같게
        self.parent[ry] = rx
        if self.rank[rx] == self.rank[ry]:
            self.rank[rx] += 1    # 높이가 같을 때만 rank 증가
```

---

## 크기 기반 Union (Union by Size)

랭크 대신 **집합의 원소 개수(size)**를 기준으로 작은 트리를 큰 트리 아래에 붙이는 방식이다. 실무에서는 랭크보다 size 기반이 더 직관적으로 쓰이는 경우가 많다.

```python
class UnionFind:
    def __init__(self, n):
        self.parent = list(range(n))
        self.size = [1] * n

    def find(self, x):
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])
        return self.parent[x]

    def union(self, x, y):
        rx, ry = self.find(x), self.find(y)
        if rx == ry:
            return
        if self.size[rx] < self.size[ry]:
            rx, ry = ry, rx
        self.parent[ry] = rx
        self.size[rx] += self.size[ry]

    def same(self, x, y):
        return self.find(x) == self.find(y)
```

---

## 시간 복잡도

경로 압축 + 랭크(또는 크기) 기반 Union을 함께 쓰면 사실상 상수 시간에 근접한다.

| 최적화 방법 | Find 시간복잡도 |
|---|---|
| 없음 (순수 재귀) | O(N) 최악 |
| 경로 압축만 | O(log N) 평균 |
| 랭크/크기 Union만 | O(log N) |
| 경로 압축 + 랭크/크기 Union | O(α(N)) ≈ O(1) |

α(N)은 아커만 함수의 역함수로, 실용적인 N 범위에서 5를 넘지 않는다. 사실상 O(1)이라고 봐도 무방하다.

---

## 활용 예시

### 1. 그래프 사이클 감지

간선 (u, v)를 추가하기 전에 `find(u) == find(v)`이면 이미 같은 집합이므로 사이클이 생긴다.

```python
uf = UnionFind(5)
edges = [(0,1), (1,2), (2,3), (3,0)]  # 마지막 간선이 사이클 형성

for u, v in edges:
    if uf.same(u, v):
        print(f"사이클 감지: {u} - {v}")
        break
    uf.union(u, v)
```

### 2. 크루스칼(Kruskal) MST

간선을 가중치 순으로 정렬 후 사이클을 만들지 않는 간선만 선택하는 알고리즘이 크루스칼이다. 사이클 판별에 유니온 파인드를 사용한다.

```python
def kruskal(n, edges):
    edges.sort(key=lambda e: e[2])  # 가중치 기준 정렬
    uf = UnionFind(n)
    mst_cost = 0

    for u, v, w in edges:
        if not uf.same(u, v):
            uf.union(u, v)
            mst_cost += w

    return mst_cost
```

### 3. 연결 컴포넌트 수 세기

모든 간선을 Union 처리한 뒤, 루트가 자기 자신인 노드의 수가 연결 컴포넌트의 수다.

```python
uf = UnionFind(6)
for u, v in [(0,1), (1,2), (3,4)]:
    uf.union(u, v)

components = sum(1 for i in range(6) if uf.find(i) == i)
print(components)  # 3 (그룹: {0,1,2}, {3,4}, {5})
```

---

## 주의할 점

**재귀 깊이 제한**: Python의 기본 재귀 제한은 1000이다. N이 큰 경우 재귀 대신 반복문으로 구현하거나 `sys.setrecursionlimit`을 늘려야 한다.

```python
def find(self, x):
    root = x
    while self.parent[root] != root:
        root = self.parent[root]
    # 경로 압축 (반복문 버전)
    while self.parent[x] != root:
        self.parent[x], x = root, self.parent[x]
    return root
```

**Union 후 Find 재사용**: Union 연산이 끝났어도 이전에 캐시된 루트 값을 직접 들고 다니면 안 된다. 항상 `find()`를 통해 현재 루트를 확인해야 한다.
