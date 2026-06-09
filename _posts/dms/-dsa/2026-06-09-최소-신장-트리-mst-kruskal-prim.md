---
layout: post
title: "[Daily morning study] 최소 신장 트리 (MST) — Kruskal과 Prim 알고리즘"
description: >
  #daily morning study
category: 
    - dms
    - dms-dsa
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 최소 신장 트리 (Minimum Spanning Tree, MST)

### 신장 트리란?

그래프 G = (V, E)에서 **신장 트리(Spanning Tree)**는 다음 조건을 만족하는 부분 그래프다.

- 모든 정점(V)을 포함
- 사이클이 없음
- 간선 수 = V - 1

즉, 모든 노드를 연결하되 사이클 없이 최소한의 간선만 사용하는 트리 구조다.

**최소 신장 트리(MST)** 는 간선에 가중치(weight)가 있을 때, 전체 가중치 합이 최소인 신장 트리다.

### 활용 사례

- 통신 네트워크 설계 (최소 비용으로 모든 도시 연결)
- 전력망, 도로망 설계
- 클러스터링 알고리즘
- 근사 알고리즘 (외판원 문제 등)

---

## Kruskal 알고리즘

### 핵심 아이디어

**간선 중심** 접근법. 가중치가 작은 간선부터 하나씩 선택하되, 사이클을 형성하는 간선은 버린다.

### 동작 과정

1. 모든 간선을 가중치 오름차순으로 정렬
2. 간선을 순서대로 꺼냄
3. 해당 간선이 사이클을 만들지 않으면 MST에 추가
4. 간선 수가 V - 1개가 되면 종료

사이클 감지에는 **Union-Find(서로소 집합)** 자료구조를 사용한다.

### Union-Find 복습

```python
parent = list(range(n))
rank = [0] * n

def find(x):
    if parent[x] != x:
        parent[x] = find(parent[x])  # 경로 압축
    return parent[x]

def union(x, y):
    rx, ry = find(x), find(y)
    if rx == ry:
        return False  # 이미 같은 집합 → 사이클
    if rank[rx] < rank[ry]:
        rx, ry = ry, rx
    parent[ry] = rx
    if rank[rx] == rank[ry]:
        rank[rx] += 1
    return True
```

### Kruskal 구현

```python
def kruskal(n, edges):
    # edges: [(weight, u, v), ...]
    edges.sort()
    parent = list(range(n))
    rank = [0] * n

    def find(x):
        if parent[x] != x:
            parent[x] = find(parent[x])
        return parent[x]

    def union(x, y):
        rx, ry = find(x), find(y)
        if rx == ry:
            return False
        if rank[rx] < rank[ry]:
            rx, ry = ry, rx
        parent[ry] = rx
        if rank[rx] == rank[ry]:
            rank[rx] += 1
        return True

    mst_weight = 0
    mst_edges = []

    for w, u, v in edges:
        if union(u, v):
            mst_weight += w
            mst_edges.append((u, v, w))
            if len(mst_edges) == n - 1:
                break

    return mst_weight, mst_edges
```

### 시간 복잡도

| 단계 | 복잡도 |
|------|--------|
| 간선 정렬 | O(E log E) |
| Union-Find 연산 | O(E α(V)) ≈ O(E) |
| 전체 | **O(E log E)** |

α는 역 아커만 함수로 사실상 상수에 가깝다.

---

## Prim 알고리즘

### 핵심 아이디어

**정점 중심** 접근법. 임의의 시작 정점에서 출발해 MST를 점진적으로 확장한다. 항상 현재 MST와 연결된 간선 중 가중치가 가장 작은 것을 선택한다.

### 동작 과정

1. 임의의 시작 정점을 MST에 추가
2. MST에 속한 정점과 연결된 모든 간선을 우선순위 큐에 삽입
3. 큐에서 최소 가중치 간선을 꺼냄
4. 목적지 정점이 아직 MST에 없으면 추가하고, 해당 정점의 간선들을 큐에 삽입
5. 모든 정점이 MST에 포함될 때까지 반복

### Prim 구현 (우선순위 큐 사용)

```python
import heapq

def prim(n, graph):
    # graph: {u: [(w, v), ...]}
    visited = [False] * n
    min_heap = [(0, 0)]  # (weight, vertex)
    mst_weight = 0

    while min_heap:
        w, u = heapq.heappop(min_heap)
        if visited[u]:
            continue
        visited[u] = True
        mst_weight += w

        for edge_w, v in graph[u]:
            if not visited[v]:
                heapq.heappush(min_heap, (edge_w, v))

    return mst_weight
```

### 시간 복잡도

| 자료구조 | 복잡도 |
|---------|--------|
| 인접 행렬 + 선형 탐색 | O(V²) |
| 인접 리스트 + 이진 힙 | O(E log V) |
| 피보나치 힙 | O(E + V log V) |

일반적으로 이진 힙을 사용한 **O(E log V)** 구현이 표준이다.

---

## Kruskal vs Prim 비교

| 비교 항목 | Kruskal | Prim |
|----------|---------|------|
| 접근 방식 | 간선 중심 | 정점 중심 |
| 핵심 자료구조 | Union-Find | 우선순위 큐 |
| 시간 복잡도 | O(E log E) | O(E log V) |
| 유리한 경우 | 희소 그래프 (E ≈ V) | 밀집 그래프 (E ≈ V²) |
| 구현 난이도 | Union-Find만 알면 간단 | 힙 관리 필요 |

- **희소 그래프**: 간선 수가 적으면 Kruskal이 정렬 후 빠르게 처리 가능
- **밀집 그래프**: 간선 수가 많으면 Prim이 유리 (V² 행렬 방식 O(V²)이 더 빠를 수도 있음)

---

## MST의 성질

**컷 속성(Cut Property):** 그래프를 두 집합으로 나누는 컷에서, 컷을 가로지르는 가장 가벼운 간선은 반드시 어떤 MST에 속한다.

이 성질이 Kruskal과 Prim 모두의 정당성(correctness)을 보장한다.

**사이클 속성(Cycle Property):** 사이클에서 가장 무거운 간선은 어떤 MST에도 속하지 않는다.

---

## 예제 풀이

```
정점: 0, 1, 2, 3, 4
간선:
  0-1 (2)
  0-3 (6)
  1-2 (3)
  1-3 (8)
  1-4 (5)
  2-4 (7)
  3-4 (9)
```

**Kruskal 순서:**
1. (2) 0-1 → 추가 ✓
2. (3) 1-2 → 추가 ✓
3. (5) 1-4 → 추가 ✓
4. (6) 0-3 → 추가 ✓
5. 간선 수 = 4 = V-1 → 종료

**MST 가중치 합 = 2 + 3 + 5 + 6 = 16**

---

## 정리

- MST는 모든 노드를 최소 비용으로 연결하는 트리
- Kruskal: 간선 정렬 후 사이클 없이 선택 → Union-Find로 구현
- Prim: 정점 확장 방식 → 우선순위 큐로 구현
- 두 알고리즘 모두 그리디(Greedy) 전략이며, 컷 속성에 의해 최적임이 보장됨
- 희소 그래프는 Kruskal, 밀집 그래프는 Prim이 일반적으로 유리
