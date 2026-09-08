---
layout: post
title: "[Daily morning study] 모노토닉 스택 (Monotonic Stack) 개념과 활용"
description: >
  #daily morning study
category: 
    - dms
    - dms-dsa
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 모노토닉 스택이란

모노토닉 스택(Monotonic Stack)은 스택 내부의 원소가 항상 단조 증가 혹은 단조 감소 순서를 유지하도록 관리하는 스택이다.

배열을 한 번 순회하면서 각 원소에 대해 "다음으로 큰 원소", "이전으로 작은 원소" 같은 질문에 O(n) 시간 안에 답하는 데 사용된다. 일반적인 이중 루프로 풀면 O(n²)이 걸리는 문제들을 선형 시간으로 줄여준다.

## 핵심 아이디어

스택에 원소를 넣기 전에 현재 원소보다 불필요해진 원소를 pop한다. 어떤 원소를 pop하느냐에 따라 두 종류로 나뉜다.

| 종류 | 스택 내 순서 | 쓰임 |
|------|------------|------|
| Monotonic Increasing Stack | 아래 → 위로 증가 | 이전/다음으로 **작은** 원소 |
| Monotonic Decreasing Stack | 아래 → 위로 감소 | 이전/다음으로 **큰** 원소 |

## 다음으로 큰 원소 (Next Greater Element)

배열에서 각 원소의 오른쪽에 있는 첫 번째로 큰 원소를 구한다.

```python
def next_greater(arr):
    n = len(arr)
    result = [-1] * n
    stack = []  # 인덱스를 저장

    for i in range(n):
        # 현재 원소가 스택 top보다 크면 → top의 NGE는 현재 원소
        while stack and arr[stack[-1]] < arr[i]:
            idx = stack.pop()
            result[idx] = arr[i]
        stack.append(i)

    return result

# 예시
arr = [4, 5, 2, 10, 8]
print(next_greater(arr))  # [5, 10, 10, -1, -1]
```

스택에는 아직 NGE를 찾지 못한 인덱스가 쌓인다. 현재 원소가 더 크면 그 인덱스의 NGE가 결정된다.

## 이전으로 작은 원소 (Previous Smaller Element)

왼쪽에서 현재 원소보다 처음으로 작은 원소를 구한다.

```python
def prev_smaller(arr):
    n = len(arr)
    result = [-1] * n
    stack = []

    for i in range(n):
        # 현재 원소보다 크거나 같은 원소는 필요 없음
        while stack and arr[stack[-1]] >= arr[i]:
            stack.pop()
        if stack:
            result[i] = arr[stack[-1]]
        stack.append(i)

    return result

arr = [3, 1, 4, 1, 5, 9, 2, 6]
print(prev_smaller(arr))  # [-1, -1, 1, -1, 1, 2, 1, 2]
```

## 대표 문제: 히스토그램에서 가장 큰 직사각형

각 막대의 높이가 주어졌을 때 히스토그램 안에서 최대 직사각형 넓이를 구한다.

### 접근

각 막대 `i`를 기준으로, `i`를 포함하는 직사각형의 너비를 결정한다.

- 왼쪽 경계: `i`의 왼쪽에서 처음으로 낮은 막대의 위치
- 오른쪽 경계: `i`의 오른쪽에서 처음으로 낮은 막대의 위치

이 두 정보를 각각 PSE, NSE(Next Smaller Element)로 구하면 O(n)에 풀린다.

```python
def largest_rectangle(heights):
    n = len(heights)
    left = [0] * n   # 왼쪽 경계 (PSE)
    right = [n] * n  # 오른쪽 경계 (NSE)
    stack = []

    # PSE
    for i in range(n):
        while stack and heights[stack[-1]] >= heights[i]:
            stack.pop()
        left[i] = stack[-1] + 1 if stack else 0
        stack.append(i)

    stack.clear()

    # NSE
    for i in range(n - 1, -1, -1):
        while stack and heights[stack[-1]] >= heights[i]:
            stack.pop()
        right[i] = stack[-1] if stack else n
        stack.append(i)

    # 최대 넓이
    max_area = 0
    for i in range(n):
        area = heights[i] * (right[i] - left[i])
        max_area = max(max_area, area)

    return max_area

heights = [2, 1, 5, 6, 2, 3]
print(largest_rectangle(heights))  # 10
```

위 예시에서 높이 5, 6인 두 막대가 만드는 2×5 = 10이 최대다.

## 다른 활용 예시

### 주식 가격 문제

`prices[i]`에서 시작해서 가격이 처음으로 내려가는 날까지의 일수를 구한다. 이 역시 NSE 변형이다.

```python
def stock_span(prices):
    n = len(prices)
    result = [0] * n
    stack = []  # 인덱스

    for i in range(n):
        while stack and prices[stack[-1]] <= prices[i]:
            stack.pop()
        result[i] = i - stack[-1] if stack else i + 1
        stack.append(i)

    return result
```

### 빗물 고이기 (Trapping Rain Water)

각 인덱스에 고이는 물의 양을 구할 때도 모노토닉 스택을 쓴다. 왼쪽 벽, 현재 바닥, 오른쪽 벽을 이용해 물의 높이와 너비를 계산한다.

```python
def trap(height):
    stack = []
    water = 0

    for i in range(len(height)):
        while stack and height[stack[-1]] < height[i]:
            bottom = stack.pop()
            if not stack:
                break
            left = stack[-1]
            h = min(height[left], height[i]) - height[bottom]
            w = i - left - 1
            water += h * w
        stack.append(i)

    return water
```

## 모노토닉 큐 (Monotonic Deque)

슬라이딩 윈도우 안에서 최솟값/최댓값을 O(1)에 구할 때 사용하는 변형이다. deque의 앞에서 윈도우를 벗어난 원소를 버리고, 뒤에서 단조성을 위반하는 원소를 제거한다.

```python
from collections import deque

def sliding_window_max(nums, k):
    dq = deque()  # 인덱스 저장, 단조 감소
    result = []

    for i in range(len(nums)):
        # 윈도우 범위를 벗어난 인덱스 제거
        if dq and dq[0] < i - k + 1:
            dq.popleft()

        # 현재보다 작거나 같은 원소는 뒤에서 제거
        while dq and nums[dq[-1]] <= nums[i]:
            dq.pop()

        dq.append(i)

        if i >= k - 1:
            result.append(nums[dq[0]])

    return result

nums = [1, 3, -1, -3, 5, 3, 6, 7]
print(sliding_window_max(nums, 3))  # [3, 3, 5, 5, 6, 7]
```

## 시간/공간 복잡도

| 연산 | 시간 복잡도 | 이유 |
|------|------------|------|
| 배열 전체 처리 | O(n) | 각 원소는 정확히 한 번 push, 한 번 pop |
| 공간 복잡도 | O(n) | 스택/deque가 최대 n개 원소 보관 |

각 원소가 스택에 들어가고 나오는 횟수가 합쳐서 O(n)이기 때문에 전체 루프 내 while 반복 횟수의 합도 O(n)이다.

## 패턴 인식 팁

다음 조건 중 하나라도 맞으면 모노토닉 스택을 의심해볼 수 있다.

- "다음으로 크거나 작은 원소" 질문
- "왼쪽/오른쪽 경계"를 구하는 문제
- 히스토그램, 건물 실루엣, 빗물 고이기처럼 넓이/높이 계산
- 슬라이딩 윈도우 내 최솟값/최댓값 (모노토닉 큐)

보통 스택에 인덱스를 저장하는 것이 값을 저장하는 것보다 유연하게 쓸 수 있다.
