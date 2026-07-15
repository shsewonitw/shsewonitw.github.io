---
layout: post
title: "[Daily morning study] 세그먼트 트리 (Segment Tree) 개념과 구현"
description: >
  #daily morning study
category: 
    - dms
    - dms-dsa
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 세그먼트 트리란?

세그먼트 트리는 배열의 구간(segment)에 대한 쿼리를 빠르게 처리하기 위한 트리 자료구조다. 주로 **구간 합**, **구간 최솟값/최댓값** 같은 연산을 O(log N) 시간에 처리할 수 있다.

### 왜 필요한가?

배열에서 구간 쿼리를 처리하는 방법은 두 가지가 있다.

| 방법 | 구간 쿼리 | 값 업데이트 |
|------|-----------|------------|
| 단순 반복 | O(N) | O(1) |
| 누적 합(Prefix Sum) | O(1) | O(N) |
| 세그먼트 트리 | O(log N) | O(log N) |

업데이트와 쿼리가 모두 빈번하게 발생하는 경우, 세그먼트 트리가 가장 효율적이다.

---

## 구조

세그먼트 트리는 완전 이진 트리 형태로, 각 노드는 특정 구간의 정보를 저장한다.

- **리프 노드**: 배열의 각 원소
- **내부 노드**: 자식 노드들의 구간을 합친 정보

배열 `[1, 3, 5, 7, 9, 11]`의 구간 합 세그먼트 트리 구조:

```
              36 [0,5]
           /           \
       9 [0,2]        27 [3,5]
       /    \          /    \
   4 [0,1]  5[2]   16[3,4]  11[5]
   /    \           /    \
1[0]   3[1]      7[3]   9[4]
```

배열 크기 N에 대해 세그먼트 트리는 보통 `4 * N` 크기의 배열로 구현한다.

---

## 구현 (Python)

### 구간 합 세그먼트 트리

```python
class SegmentTree:
    def __init__(self, arr):
        self.n = len(arr)
        self.tree = [0] * (4 * self.n)
        self.build(arr, 1, 0, self.n - 1)

    def build(self, arr, node, start, end):
        if start == end:
            self.tree[node] = arr[start]
        else:
            mid = (start + end) // 2
            self.build(arr, 2 * node, start, mid)
            self.build(arr, 2 * node + 1, mid + 1, end)
            self.tree[node] = self.tree[2 * node] + self.tree[2 * node + 1]

    def update(self, node, start, end, idx, val):
        if start == end:
            self.tree[node] = val
        else:
            mid = (start + end) // 2
            if idx <= mid:
                self.update(2 * node, start, mid, idx, val)
            else:
                self.update(2 * node + 1, mid + 1, end, idx, val)
            self.tree[node] = self.tree[2 * node] + self.tree[2 * node + 1]

    def query(self, node, start, end, left, right):
        # 현재 구간이 쿼리 범위를 벗어남
        if right < start or end < left:
            return 0
        # 현재 구간이 쿼리 범위 안에 완전히 포함됨
        if left <= start and end <= right:
            return self.tree[node]
        mid = (start + end) // 2
        left_sum = self.query(2 * node, start, mid, left, right)
        right_sum = self.query(2 * node + 1, mid + 1, end, left, right)
        return left_sum + right_sum

    def range_sum(self, left, right):
        return self.query(1, 0, self.n - 1, left, right)

    def point_update(self, idx, val):
        self.update(1, 0, self.n - 1, idx, val)


# 사용 예시
arr = [1, 3, 5, 7, 9, 11]
st = SegmentTree(arr)

print(st.range_sum(1, 3))  # 3+5+7 = 15
st.point_update(1, 10)     # arr[1] = 10
print(st.range_sum(1, 3))  # 10+5+7 = 22
```

---

## 핵심 연산 흐름

### 빌드 (Build)

리프 노드부터 위로 올라가면서 부모 노드를 채운다. 시간 복잡도 O(N).

### 업데이트 (Update)

변경할 인덱스가 포함된 경로를 따라 루트까지 올라가며 값을 갱신한다.

```
arr[1] 값 변경 시 영향을 받는 노드:
[0,5] → [0,2] → [0,1] → [1]
```

루트까지의 높이가 O(log N)이므로 업데이트도 O(log N).

### 쿼리 (Query)

구간이 현재 노드의 범위를 벗어나면 0(합산 무관 값) 반환, 완전히 포함되면 현재 노드 값 반환, 걸쳐 있으면 재귀적으로 분할해 처리한다.

---

## 구간 최솟값 변형

구간 합 대신 최솟값을 저장하도록 수정하면 RMQ(Range Minimum Query) 트리가 된다. 병합 연산만 바꿔주면 된다.

```python
# 빌드 & 업데이트 병합 연산 변경
self.tree[node] = min(self.tree[2 * node], self.tree[2 * node + 1])

# 쿼리 기저 조건 변경
if right < start or end < left:
    return float('inf')  # 합의 0 대신 INF 반환
```

---

## 시간 복잡도 정리

| 연산 | 시간 복잡도 |
|------|------------|
| 트리 빌드 | O(N) |
| 포인트 업데이트 | O(log N) |
| 구간 쿼리 | O(log N) |
| 공간 복잡도 | O(N) |

---

## 활용 사례

- **구간 합 쿼리**: 연속된 배열 구간의 합
- **구간 최솟값/최댓값 (RMQ)**: 특정 구간 내 최솟값/최댓값 탐색
- **구간 GCD**: 특정 구간의 최대공약수
- **레이지 프로파게이션(Lazy Propagation)**: 구간 업데이트가 필요한 경우 (예: 특정 구간의 모든 값에 +k)

---

## 레이지 프로파게이션 (Lazy Propagation) 개요

구간 업데이트(range update)까지 지원하려면 레이지 배열을 추가한다. 업데이트를 즉시 반영하지 않고, 해당 노드에 "나중에 처리할 변경값"을 저장해 두다가, 쿼리나 하위 노드 접근 시 실제로 반영한다.

```
[0,5] 구간에 +3 업데이트
→ 루트에 lazy[1] = 3 저장
→ 실제 자식에게는 필요할 때만 전파 (push down)
```

기본 세그먼트 트리의 구간 업데이트는 O(N)이지만, 레이지를 쓰면 O(log N)으로 줄어든다.

---

## 세그먼트 트리 vs 다른 자료구조

| 자료구조 | 포인트 업데이트 | 구간 쿼리 | 구간 업데이트 |
|---------|--------------|----------|--------------|
| 배열 | O(1) | O(N) | O(N) |
| 누적 합 | O(N) | O(1) | O(N) |
| 펜윅 트리 (BIT) | O(log N) | O(log N) | O(log N) |
| 세그먼트 트리 | O(log N) | O(log N) | O(N) / O(log N)* |

*레이지 프로파게이션 적용 시 O(log N)

펜윅 트리(Binary Indexed Tree)는 세그먼트 트리보다 구현이 간단하고 상수 배가 작지만, 세그먼트 트리가 지원하는 모든 연산 유형을 처리하지는 못한다. 복잡한 구간 연산이 필요하면 세그먼트 트리가 더 유연하다.
