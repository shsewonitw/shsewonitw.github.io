---
layout: post
title: "[Daily morning study] 최장 증가 부분수열 (LIS, Longest Increasing Subsequence)"
description: >
  #daily morning study
category: 
    - dms
    - dms-dsa
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## LIS란?

수열에서 **원소를 일부 선택해 만든 부분 수열 중, 각 원소가 이전 원소보다 엄격히 크면서 가장 긴 것**을 최장 증가 부분수열(LIS)이라고 한다.

예시: `[3, 1, 4, 1, 5, 9, 2, 6]`

이 수열에서 LIS는 `[1, 4, 5, 9]` 또는 `[1, 4, 5, 6]` 등 여러 개일 수 있고, 길이는 **4**다.

핵심 포인트:
- 선택한 원소의 **위치(인덱스)**는 앞에서 뒤로 순서를 유지해야 한다 (부분수열이므로)
- 값은 엄격히 증가해야 한다 (`a[i] < a[j]`, `i < j`)
- 같은 길이의 LIS가 여러 개 존재할 수 있다

---

## O(n²) DP 풀이

### 점화식

`dp[i]` = i번째 원소를 마지막으로 포함하는 LIS의 길이

```
dp[i] = max(dp[j] + 1)  for all j < i where arr[j] < arr[i]
dp[i] = 1  (초기값: 자기 자신만으로 이루어진 LIS)
```

### 구현

```python
def lis_dp(arr):
    n = len(arr)
    dp = [1] * n

    for i in range(1, n):
        for j in range(i):
            if arr[j] < arr[i]:
                dp[i] = max(dp[i], dp[j] + 1)

    return max(dp)

arr = [3, 1, 4, 1, 5, 9, 2, 6]
print(lis_dp(arr))  # 4
```

### 과정 추적

| i | arr[i] | 비교 대상 | dp[i] |
|---|--------|-----------|-------|
| 0 | 3      | 없음      | 1     |
| 1 | 1      | 없음      | 1     |
| 2 | 4      | 3, 1      | 2     |
| 3 | 1      | 없음      | 1     |
| 4 | 5      | 3, 1, 4, 1 | 3   |
| 5 | 9      | 3,1,4,1,5  | 4   |
| 6 | 2      | 1          | 2   |
| 7 | 6      | 3,1,4,1,5,2| 4   |

### 시간/공간 복잡도

- 시간: O(n²) - 이중 반복문
- 공간: O(n) - dp 배열

n이 작을 때 (n ≤ 1000~2000)는 충분하지만, n이 크면 이분 탐색 최적화가 필요하다.

---

## O(n log n) 이분 탐색 최적화

### 핵심 아이디어

LIS의 길이를 구하기 위해 **인내심 정렬(Patience Sorting)**의 개념을 활용한다.

`tails` 배열을 유지하는데, `tails[i]`는 **길이가 i+1인 증가 부분수열의 마지막 원소 중 가장 작은 값**을 저장한다.

왜 이렇게 하는가? 마지막 원소가 작을수록 이후 원소를 붙일 가능성이 높기 때문이다.

### 알고리즘

각 원소 `x`에 대해:
1. `tails`에서 `x` 이상인 첫 번째 위치를 이분 탐색으로 찾는다 (`lower_bound`)
2. 해당 위치에 `x`를 삽입(덮어씀)한다
3. `x`가 `tails`의 모든 원소보다 크면 뒤에 추가한다

`tails`의 최종 길이가 LIS의 길이다.

### 구현

```python
import bisect

def lis_binary_search(arr):
    tails = []

    for x in arr:
        pos = bisect.bisect_left(tails, x)  # x 이상인 첫 번째 위치
        if pos == len(tails):
            tails.append(x)
        else:
            tails[pos] = x

    return len(tails)

arr = [3, 1, 4, 1, 5, 9, 2, 6]
print(lis_binary_search(arr))  # 4
```

### 과정 추적

| 처리 원소 | tails 변화     | 설명                          |
|----------|---------------|-------------------------------|
| 3        | [3]           | 빈 배열이므로 추가              |
| 1        | [1]           | 3 ≥ 1, 위치 0에 덮어씀         |
| 4        | [1, 4]        | 4 > 1, 뒤에 추가               |
| 1        | [1, 4]        | 1 ≥ 1, 위치 0에 덮어씀(변화없음) |
| 5        | [1, 4, 5]     | 5 > 4, 뒤에 추가               |
| 9        | [1, 4, 5, 9]  | 9 > 5, 뒤에 추가               |
| 2        | [1, 2, 5, 9]  | 2 < 4, 위치 1에 덮어씀         |
| 6        | [1, 2, 5, 6]  | 6 < 9, 위치 3에 덮어씀         |

최종 `tails` = `[1, 2, 5, 6]`, 길이 = **4**

**주의**: `tails` 자체가 실제 LIS는 아니다. 길이만 정확하다. 실제 LIS를 구성하려면 별도의 parent 추적이 필요하다.

### 시간/공간 복잡도

- 시간: O(n log n) - n번 순회 × 이분 탐색 log n
- 공간: O(n) - tails 배열

---

## 실제 LIS 원소 추적

길이만이 아니라 LIS를 구성하는 실제 원소를 알고 싶을 때는 각 원소가 삽입된 위치를 기록해야 한다.

```python
import bisect

def lis_with_elements(arr):
    n = len(arr)
    tails = []
    pos = [0] * n      # 각 원소가 tails의 어느 위치에 삽입됐는지
    parent = [-1] * n  # 이전 원소의 인덱스

    for i, x in enumerate(arr):
        p = bisect.bisect_left(tails, x)
        pos[i] = p
        if p == len(tails):
            tails.append(x)
        else:
            tails[p] = x

    # 역추적
    lis_len = len(tails)
    result = []
    cur_pos = lis_len - 1

    for i in range(n - 1, -1, -1):
        if pos[i] == cur_pos:
            result.append(arr[i])
            cur_pos -= 1
            if cur_pos < 0:
                break

    return result[::-1]

arr = [3, 1, 4, 1, 5, 9, 2, 6]
print(lis_with_elements(arr))  # [1, 4, 5, 6] 또는 [1, 4, 5, 9]
```

---

## 변형 문제들

### 비감소 LIS (non-decreasing)

같은 값도 허용하는 경우: `bisect_left` 대신 `bisect_right` 사용

```python
pos = bisect.bisect_right(tails, x)
```

### 최장 감소 부분수열 (LDS)

배열을 뒤집어서 LIS를 구하면 된다.

```python
def lds(arr):
    return lis_binary_search(arr[::-1])
```

### 최장 바이토닉 부분수열

왼쪽에서의 LIS 길이와 오른쪽에서의 LIS 길이(= 왼쪽에서의 LDS)를 합산해 구한다.

```python
def lbs(arr):
    n = len(arr)
    lis = [1] * n
    lds = [1] * n

    # 앞에서 뒤로 LIS
    for i in range(1, n):
        for j in range(i):
            if arr[j] < arr[i]:
                lis[i] = max(lis[i], lis[j] + 1)

    # 뒤에서 앞으로 LIS (= LDS)
    for i in range(n - 2, -1, -1):
        for j in range(i + 1, n):
            if arr[j] < arr[i]:
                lds[i] = max(lds[i], lds[j] + 1)

    return max(lis[i] + lds[i] - 1 for i in range(n))
```

---

## 정리

| 방법           | 시간 복잡도 | 공간 복잡도 | 특징                              |
|---------------|------------|------------|-----------------------------------|
| O(n²) DP      | O(n²)      | O(n)       | 구현 단순, n ≤ 2000 적합           |
| O(n log n) BS | O(n log n) | O(n)       | 대용량 입력 처리, n ≤ 100,000 적합 |

- LIS 길이만 필요하면 이분 탐색 버전을 쓰는 게 훨씬 효율적이다
- 실제 LIS를 구성해야 하면 parent 배열로 역추적이 필요하다
- 비감소 / 비증가 조건에 따라 `bisect_left` / `bisect_right` 를 구분해서 써야 한다
- 코딩 테스트에서 자주 나오는 패턴이므로 두 방법 모두 손으로 구현할 수 있어야 한다
