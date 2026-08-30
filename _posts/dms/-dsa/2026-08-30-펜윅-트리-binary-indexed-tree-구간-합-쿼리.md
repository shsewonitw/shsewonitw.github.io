---
layout: post
title: "[Daily morning study] 펜윅 트리 (Binary Indexed Tree) — 구간 합 쿼리"
description: >
  #daily morning study
category: 
    - dms
    - dms-dsa
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 펜윅 트리 (Fenwick Tree / Binary Indexed Tree)

### 왜 필요한가?

배열에서 특정 구간의 합을 자주 구해야 하고, 동시에 원소 값도 자주 바뀌는 상황을 생각해보자.

| 방법 | 구간 합 조회 | 원소 업데이트 |
|---|---|---|
| 단순 배열 | O(N) | O(1) |
| 누적 합(Prefix Sum) | O(1) | O(N) |
| 세그먼트 트리 | O(log N) | O(log N) |
| **펜윅 트리** | **O(log N)** | **O(log N)** |

세그먼트 트리와 같은 시간 복잡도지만, 펜윅 트리는 구현이 더 단순하고 메모리도 절반 정도만 쓴다. 구간 합 + 점 업데이트 조합에서 가장 실용적인 선택이다.

---

### 핵심 아이디어: 비트 연산

펜윅 트리는 인덱스의 이진 표현(binary representation)에서 마지막 1 비트를 이용한다.

어떤 인덱스 `i`에 대해 **`i & (-i)`** 를 계산하면 마지막 1 비트만 남는다.

```
i = 6  →  0110
-i     →  1010  (2의 보수)
i&(-i) →  0010  →  2
```

이 값을 **`lowbit(i)`** 라고 부른다.

---

### 트리 구조

BIT 배열 `bit[i]`는 `i`에서 `lowbit(i)` 개의 원소 합을 저장한다.

```
인덱스   이진   lowbit  담당 범위
  1    0001     1      [1, 1]
  2    0010     2      [1, 2]
  3    0011     1      [3, 3]
  4    0100     4      [1, 4]
  5    0101     1      [5, 5]
  6    0110     2      [5, 6]
  7    0111     1      [7, 7]
  8    1000     8      [1, 8]
```

---

### 구현: 1-indexed 기준

```python
class BIT:
    def __init__(self, n):
        self.n = n
        self.bit = [0] * (n + 1)  # 1-indexed

    # i번째 원소에 delta 더하기
    def update(self, i, delta):
        while i <= self.n:
            self.bit[i] += delta
            i += i & (-i)  # 부모 노드로 이동

    # [1, i] 구간의 합
    def query(self, i):
        s = 0
        while i > 0:
            s += self.bit[i]
            i -= i & (-i)  # 이전 구간으로 이동
        return s

    # [l, r] 구간의 합
    def range_query(self, l, r):
        return self.query(r) - self.query(l - 1)
```

#### update 동작 추적

`update(3, 5)` 호출 시:

```
i = 3  →  bit[3] += 5  →  i += lowbit(3) = 1  →  i = 4
i = 4  →  bit[4] += 5  →  i += lowbit(4) = 4  →  i = 8
i = 8  →  bit[8] += 5  →  i += lowbit(8) = 8  →  i = 16 > n, 종료
```

#### query 동작 추적

`query(7)` (즉, [1,7] 합) 호출 시:

```
i = 7  →  s += bit[7]  →  i -= lowbit(7) = 1  →  i = 6
i = 6  →  s += bit[6]  →  i -= lowbit(6) = 2  →  i = 4
i = 4  →  s += bit[4]  →  i -= lowbit(4) = 4  →  i = 0, 종료
```

bit[7] + bit[6] + bit[4] = [7,7] + [5,6] + [1,4] = [1,7] 합. 완벽하게 겹치지 않는 구간들을 합산한다.

---

### 초기화: 배열에서 BIT 구성

방법 1 — update 반복 호출: O(N log N)

```python
def build(self, arr):
    for i, v in enumerate(arr, 1):
        self.update(i, v)
```

방법 2 — O(N) 선형 구성

```python
def build_linear(self, arr):
    for i, v in enumerate(arr, 1):
        self.bit[i] += v
        j = i + (i & (-i))
        if j <= self.n:
            self.bit[j] += self.bit[i]
```

---

### 활용 예시

#### 구간 합 + 점 업데이트

```python
arr = [3, 2, -1, 6, 5, 4, -3, 3]
n = len(arr)
bit = BIT(n)
bit.build(arr)

print(bit.range_query(2, 6))  # arr[1..5] = 2+(-1)+6+5+4 = 16

arr[3] = 10  # 인덱스 4 (1-indexed)
bit.update(4, 10 - (-1))  # 기존 값과의 차이

print(bit.range_query(2, 6))  # 2+10+6+5+4 = 27
```

#### 역전쌍(Inversion Count) 세기

배열에서 `i < j`이고 `arr[i] > arr[j]`인 쌍의 수를 구하는 문제에 BIT를 쓸 수 있다.

좌표 압축 후, 왼쪽부터 처리하며 현재 원소보다 큰 값이 이미 삽입된 개수를 BIT로 조회한다.

```python
def count_inversions(arr):
    n = len(arr)
    # 좌표 압축
    sorted_arr = sorted(set(arr))
    rank = {v: i+1 for i, v in enumerate(sorted_arr)}
    
    bit = BIT(len(sorted_arr))
    inversions = 0
    
    for v in arr:
        r = rank[v]
        # r보다 큰 값 중 이미 처리된 것들의 수 = 역전쌍
        inversions += bit.query(len(sorted_arr)) - bit.query(r)
        bit.update(r, 1)
    
    return inversions

print(count_inversions([3, 1, 2]))  # 2 (3>1, 3>2)
```

---

### 2D BIT (2차원 구간 합)

2차원 배열에서 부분 행렬 합도 BIT로 처리할 수 있다.

```python
class BIT2D:
    def __init__(self, n, m):
        self.n, self.m = n, m
        self.bit = [[0] * (m + 1) for _ in range(n + 1)]

    def update(self, r, c, delta):
        i = r
        while i <= self.n:
            j = c
            while j <= self.m:
                self.bit[i][j] += delta
                j += j & (-j)
            i += i & (-i)

    def query(self, r, c):
        s = 0
        i = r
        while i > 0:
            j = c
            while j > 0:
                s += self.bit[i][j]
                j -= j & (-j)
            i -= i & (-i)
        return s

    # (r1,c1)~(r2,c2) 부분 행렬 합
    def range_query(self, r1, c1, r2, c2):
        return (self.query(r2, c2)
                - self.query(r1 - 1, c2)
                - self.query(r2, c1 - 1)
                + self.query(r1 - 1, c1 - 1))
```

시간 복잡도: 업데이트/쿼리 모두 O(log N × log M)

---

### 구간 업데이트 + 점 조회

기본 BIT는 점 업데이트 + 구간 조회지만, **차분 배열 기법**을 쓰면 구간 업데이트 + 점 조회로 뒤집을 수 있다.

차분 배열 `D[i] = A[i] - A[i-1]`를 BIT로 관리하면:

- `A[l..r]`에 `v`를 더하는 건 `D[l]`에 `+v`, `D[r+1]`에 `-v`
- `A[i]` 값을 보는 건 `query(i)` = `D[1] + ... + D[i]`

```python
# 구간 업데이트: A[l..r] += v
def range_update(bit, l, r, v):
    bit.update(l, v)
    bit.update(r + 1, -v)

# 점 조회: A[i]의 현재 값
def point_query(bit, i):
    return bit.query(i)
```

---

### BIT vs 세그먼트 트리 선택 기준

| 상황 | 선택 |
|---|---|
| 구간 합 + 점 업데이트만 필요 | BIT (더 간단) |
| 구간 최솟값/최댓값 쿼리 | 세그먼트 트리 |
| 구간 업데이트 + 구간 쿼리 (lazy) | 세그먼트 트리 |
| 코딩 테스트에서 빠르게 | BIT |
| 다양한 연산 커스터마이징 | 세그먼트 트리 |

BIT는 **합(sum) 연산처럼 역원이 존재하는 경우**에만 쓸 수 있다. 최솟값/최댓값에는 쓸 수 없다.

---

### 핵심 정리

- `i & (-i)`: 인덱스 `i`의 마지막 1 비트만 추출 → BIT의 핵심 연산
- **update**: `i`에서 시작해 `i += lowbit(i)`로 올라가며 갱신
- **query**: `i`에서 시작해 `i -= lowbit(i)`로 내려오며 누적
- 시간 복잡도: 업데이트/쿼리 O(log N), 공간 O(N)
- 구간 합 + 점 업데이트 문제의 가장 단순하고 빠른 해결책
