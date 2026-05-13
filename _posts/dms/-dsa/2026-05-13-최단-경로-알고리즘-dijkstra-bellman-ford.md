---
layout: post
title: "[Daily morning study] 최단 경로 알고리즘 (Dijkstra, Bellman-Ford)"
description: >
  #daily morning study
category: 
    - dms
    - dms-dsa
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 최단 경로 문제란?

그래프에서 두 정점 사이의 경로 중 가중치 합이 가장 작은 경로를 찾는 문제다.

현실 세계 예시:
- 지도 앱에서 목적지까지 가장 빠른 길 찾기
- 네트워크 라우팅 경로 최적화
- 게임에서 NPC 이동 경로 계산

그래프 조건에 따라 적합한 알고리즘이 다르다.

| 조건 | 적합한 알고리즘 |
|------|----------------|
| 음수 가중치 없음 | Dijkstra |
| 음수 가중치 있음 | Bellman-Ford |
| 모든 쌍 최단 경로 | Floyd-Warshall |

---

## Dijkstra 알고리즘

### 핵심 아이디어

"현재까지 발견된 가장 짧은 경로를 가진 정점부터 처리한다."

탐욕적(Greedy) 접근 방식으로, 한 번 확정된 정점의 최단 거리는 변경되지 않는다.

### 동작 과정

1. 시작 정점 거리 = 0, 나머지 = ∞로 초기화
2. 방문하지 않은 정점 중 거리가 가장 짧은 정점 선택
3. 선택된 정점의 인접 정점들에 대해 거리 갱신 (relaxation)
4. 선택된 정점을 방문 처리
5. 모든 정점이 방문될 때까지 2~4 반복

### 예시 추적

```
A --1-- B --3-- D
|       |
4       2
|       |
C       E --5-- F
```

시작 정점: A

- 초기: `{A:0, B:∞, C:∞, D:∞, E:∞, F:∞}`
- A 처리 → B=1, C=4
- B 처리 (dist=1) → D=4, E=3
- E 처리 (dist=3) → F=8
- C 처리 (dist=4) → E는 이미 3이므로 갱신 안 함
- D 처리 (dist=4) → 인접 없음
- F 처리 (dist=8) → 완료

최종: `{A:0, B:1, C:4, D:4, E:3, F:8}`

### 코드 (Python, 우선순위 큐)

```python
import heapq

def dijkstra(graph, start):
    dist = {node: float('inf') for node in graph}
    dist[start] = 0
    heap = [(0, start)]  # (거리, 정점)

    while heap:
        current_dist, u = heapq.heappop(heap)

        if current_dist > dist[u]:
            continue  # 이미 더 짧은 경로로 처리된 경우 skip

        for v, weight in graph[u]:
            new_dist = dist[u] + weight
            if new_dist < dist[v]:
                dist[v] = new_dist
                heapq.heappush(heap, (new_dist, v))

    return dist

graph = {
    'A': [('B', 1), ('C', 4)],
    'B': [('D', 3), ('E', 2)],
    'C': [('E', 1)],
    'D': [],
    'E': [('F', 5)],
    'F': []
}

print(dijkstra(graph, 'A'))
# {'A': 0, 'B': 1, 'C': 4, 'D': 4, 'E': 3, 'F': 8}
```

`heapq`를 쓰는 이유: 매번 가장 거리가 짧은 정점을 O(log N)에 꺼내기 위해서다. 단순 배열로 최솟값을 찾으면 O(V)가 걸린다.

### 시간 복잡도

| 구현 방식 | 시간 복잡도 |
|----------|-------------|
| 단순 배열 | O(V²) |
| 우선순위 큐 (힙) | O((V + E) log V) |

밀집 그래프(E ≈ V²)에서는 배열이, 희소 그래프에서는 힙 구현이 유리하다.

### 주의사항

**음수 가중치가 있으면 Dijkstra는 틀린 답을 낸다.**

탐욕적 선택이 깨지기 때문이다. 한 번 확정된 거리가 나중에 음수 간선을 타고 더 작아질 수 있다.

---

## Bellman-Ford 알고리즘

### 핵심 아이디어

"모든 간선을 V-1번 반복해서 relaxation한다."

V-1번인 이유: 최단 경로에 포함될 수 있는 간선의 최대 개수가 V-1이기 때문이다.

### 동작 과정

1. 시작 정점 거리 = 0, 나머지 = ∞로 초기화
2. 모든 간선에 대해 relaxation 수행
3. 2번을 V-1번 반복
4. V번째 반복에서도 갱신이 발생하면 → 음수 사이클 존재

### 코드 (Python)

```python
def bellman_ford(vertices, edges, start):
    dist = {v: float('inf') for v in vertices}
    dist[start] = 0

    for _ in range(len(vertices) - 1):
        for u, v, weight in edges:
            if dist[u] != float('inf') and dist[u] + weight < dist[v]:
                dist[v] = dist[u] + weight

    # 음수 사이클 검사: V번째에도 갱신되면 사이클 있음
    for u, v, weight in edges:
        if dist[u] != float('inf') and dist[u] + weight < dist[v]:
            return None  # 음수 사이클 존재

    return dist

vertices = ['A', 'B', 'C', 'D']
edges = [
    ('A', 'B', 1),
    ('B', 'C', -3),
    ('C', 'D', 2),
    ('A', 'C', 4)
]

print(bellman_ford(vertices, edges, 'A'))
# {'A': 0, 'B': 1, 'C': -2, 'D': 0}
```

### 시간 복잡도

| | Bellman-Ford |
|---|---|
| 시간 복잡도 | O(V × E) |
| 공간 복잡도 | O(V) |

Dijkstra보다 느리지만 음수 가중치와 음수 사이클 감지가 가능하다.

---

## 음수 사이클이 문제인 이유

음수 사이클이 존재하면 그 사이클을 계속 돌수록 경로 비용이 줄어들어 최단 경로가 정의되지 않는다.

예) `A → B → C → A` 경로 비용 = -5  
→ 이 사이클을 1번 돌면 -5, 2번 돌면 -10, n번 돌면 -5n → 발산

Bellman-Ford는 V번째 relaxation에서도 갱신이 발생하면 음수 사이클이 있다고 판단한다.

---

## 두 알고리즘 비교

| 항목 | Dijkstra | Bellman-Ford |
|------|----------|--------------|
| 시간 복잡도 | O((V+E) log V) | O(V × E) |
| 음수 가중치 | ❌ 불가 | ✅ 가능 |
| 음수 사이클 감지 | ❌ 불가 | ✅ 가능 |
| 구현 복잡도 | 중간 (힙 사용) | 단순 |
| 적합한 상황 | 가중치 ≥ 0인 그래프 | 음수 가중치 존재 시 |

---

## 정리

- **Dijkstra**: 빠르지만 음수 가중치 불가. 실용적인 대부분의 상황에서 사용.
- **Bellman-Ford**: 느리지만 음수 가중치 처리 가능. 음수 사이클 감지도 가능.
- 두 알고리즘 모두 단일 출발점 최단 경로(Single Source Shortest Path, SSSP) 문제를 푼다.
- 모든 정점 쌍의 최단 경로가 필요하면 Floyd-Warshall(O(V³))을 사용한다.
