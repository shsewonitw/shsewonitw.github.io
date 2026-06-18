---
layout: post
title: "[Daily morning study] 위상 정렬 (Topological Sort) 개념과 활용"
description: >
  #daily morning study
category: 
    - dms
    - dms-dsa
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 위상 정렬 (Topological Sort)

### 개념

위상 정렬은 **방향 비순환 그래프(DAG, Directed Acyclic Graph)** 에서 정점들을 간선의 방향에 따라 순서대로 나열하는 알고리즘이다.

핵심 조건: **A → B 간선이 있다면, 정렬 결과에서 A는 항상 B보다 앞에 위치**해야 한다.

실생활 예시:
- 대학 수강 신청: 선수 과목을 먼저 들어야 다음 과목을 수강할 수 있음
- 빌드 시스템: 의존 모듈을 먼저 컴파일해야 함
- 작업 스케줄링: 선행 작업이 완료되어야 다음 작업 시작 가능

### DAG란?

위상 정렬이 가능하려면 반드시 **DAG** 여야 한다.

- **Directed (방향 그래프)**: 간선에 방향이 있음
- **Acyclic (비순환)**: 사이클이 없음

사이클이 있으면 "A → B → C → A" 같은 순환 의존이 생겨 순서를 정할 수 없다. 사이클 그래프에서는 위상 정렬 결과가 존재하지 않는다.

---

## 구현 방법 1: Kahn's Algorithm (BFS 기반)

### 핵심 아이디어

**진입 차수(in-degree)** 를 활용한다. 진입 차수란 특정 정점으로 들어오는 간선의 수다.

1. 모든 정점의 진입 차수를 계산한다.
2. 진입 차수가 0인 정점(선행 조건 없음)을 큐에 넣는다.
3. 큐에서 정점을 꺼내 결과 배열에 추가하고, 해당 정점에서 나가는 간선을 제거한다.
4. 간선 제거로 인해 진입 차수가 0이 된 정점을 큐에 추가한다.
5. 큐가 빌 때까지 반복한다.
6. 결과 배열의 크기가 전체 정점 수보다 작으면 **사이클이 존재**한다.

### 코드 (Python)

```python
from collections import deque

def topological_sort(n, edges):
    # n: 정점 수, edges: [(from, to), ...]
    graph = [[] for _ in range(n)]
    in_degree = [0] * n

    for u, v in edges:
        graph[u].append(v)
        in_degree[v] += 1

    queue = deque()
    for i in range(n):
        if in_degree[i] == 0:
            queue.append(i)

    result = []
    while queue:
        node = queue.popleft()
        result.append(node)

        for neighbor in graph[node]:
            in_degree[neighbor] -= 1
            if in_degree[neighbor] == 0:
                queue.append(neighbor)

    if len(result) != n:
        return []  # 사이클 존재

    return result

# 예시: 0 → 1 → 3, 0 → 2 → 3
edges = [(0, 1), (0, 2), (1, 3), (2, 3)]
print(topological_sort(4, edges))  # [0, 1, 2, 3] 또는 [0, 2, 1, 3]
```

위상 정렬 결과는 **유일하지 않다**. 진입 차수가 0인 정점이 여러 개이면 그 중 어느 것을 먼저 선택하느냐에 따라 결과가 달라진다.

---

## 구현 방법 2: DFS 기반

### 핵심 아이디어

DFS로 그래프를 탐색하되, 재귀 호출이 완전히 끝난(모든 후손 방문 완료) 정점을 **스택에 push** 한다. 마지막에 스택을 뒤집으면 위상 정렬 순서가 된다.

### 코드 (Python)

```python
def topological_sort_dfs(n, edges):
    graph = [[] for _ in range(n)]
    for u, v in edges:
        graph[u].append(v)

    visited = [False] * n
    on_stack = [False] * n  # 현재 DFS 경로상에 있는지 (사이클 탐지용)
    stack = []
    has_cycle = [False]

    def dfs(node):
        if has_cycle[0]:
            return
        visited[node] = True
        on_stack[node] = True

        for neighbor in graph[node]:
            if not visited[neighbor]:
                dfs(neighbor)
            elif on_stack[neighbor]:
                has_cycle[0] = True
                return

        on_stack[node] = False
        stack.append(node)  # 후손 탐색 완료 후 push

    for i in range(n):
        if not visited[i]:
            dfs(i)

    if has_cycle[0]:
        return []

    return stack[::-1]  # 역순이 위상 정렬 결과
```

### Kahn's vs DFS 비교

| 항목 | Kahn's Algorithm | DFS 기반 |
|------|-----------------|----------|
| 방식 | BFS (큐) | DFS (재귀/스택) |
| 사이클 탐지 | 결과 길이 확인 | on_stack 배열 |
| 구현 난이도 | 직관적 | 약간 복잡 |
| 결과 안정성 | 진입 차수 0인 순서대로 | 탐색 순서에 따라 다름 |

---

## 복잡도 분석

- **시간 복잡도**: O(V + E)
  - 모든 정점과 간선을 한 번씩 처리
- **공간 복잡도**: O(V + E)
  - 그래프 인접 리스트, 진입 차수 배열, 큐/스택

---

## 활용 사례

### 1. 선수 과목 계획

```
수학기초 → 선형대수 → 머신러닝
           ↘ 확률론 ↗
```

위상 정렬로 수강 가능한 순서를 자동으로 계산할 수 있다.

### 2. 패키지 의존성 설치 순서

npm, pip 같은 패키지 매니저가 내부적으로 사용한다. A 패키지가 B, C에 의존할 때 B, C를 먼저 설치하도록 순서를 결정한다.

### 3. 작업 병렬화

위상 정렬 결과에서 같은 레벨(진입 차수가 동시에 0이 되는 정점들)은 **병렬 실행**이 가능하다. CI/CD 파이프라인의 병렬 스텝 구성에 응용된다.

### 4. 사이클 탐지

컴파일러에서 순환 import, 순환 참조 탐지에 활용한다.

---

## 실전 문제 유형

### 유형 1: 단순 위상 정렬

"N개의 작업이 있고, 일부 작업은 다른 작업보다 먼저 수행되어야 한다. 수행 가능한 순서를 출력하라."

→ Kahn's Algorithm 직접 적용

### 유형 2: 사이클 탐지 포함

"선수 과목 관계가 주어질 때, 모든 과목을 수강할 수 있는지 판별하라."

→ 위상 정렬 후 결과 길이 != 정점 수이면 불가능

### 유형 3: 사전순 최소 위상 정렬

"가능한 위상 정렬 중 사전순으로 가장 빠른 것을 출력하라."

→ 큐 대신 **우선순위 큐(최소 힙)** 를 사용해 항상 가장 작은 번호를 먼저 선택

```python
import heapq

def topological_sort_lex(n, edges):
    graph = [[] for _ in range(n)]
    in_degree = [0] * n

    for u, v in edges:
        graph[u].append(v)
        in_degree[v] += 1

    heap = []
    for i in range(n):
        if in_degree[i] == 0:
            heapq.heappush(heap, i)

    result = []
    while heap:
        node = heapq.heappop(heap)
        result.append(node)
        for neighbor in graph[node]:
            in_degree[neighbor] -= 1
            if in_degree[neighbor] == 0:
                heapq.heappush(heap, neighbor)

    return result if len(result) == n else []
```

---

## 요약

- 위상 정렬은 DAG에서만 동작하며, 사이클이 있으면 결과가 존재하지 않는다.
- Kahn's Algorithm은 진입 차수 0인 정점부터 BFS로 처리한다.
- DFS 기반은 재귀 완료 후 스택에 push 후 역순으로 출력한다.
- 시간/공간 복잡도는 모두 O(V + E)다.
- 결과는 유일하지 않으며, 사전순 최소가 필요하면 최소 힙을 활용한다.
