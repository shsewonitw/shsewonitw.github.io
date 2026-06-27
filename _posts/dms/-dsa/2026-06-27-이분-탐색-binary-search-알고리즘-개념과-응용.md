---
layout: post
title: "[Daily morning study] 이분 탐색 (Binary Search) 알고리즘 개념과 응용"
description: >
  #daily morning study
category: 
    - dms
    - dms-dsa
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 이분 탐색 (Binary Search)

### 개념

이분 탐색은 **정렬된 배열**에서 특정 값을 찾을 때 탐색 범위를 절반씩 줄여나가는 알고리즘이다. 매 단계마다 중앙값과 목표값을 비교해 왼쪽 또는 오른쪽 절반만 살펴보므로 시간 복잡도가 O(log n)이다.

선형 탐색(O(n))과 비교하면 데이터가 많을수록 차이가 극적으로 벌어진다.

| 데이터 크기 | 선형 탐색 | 이분 탐색 |
|-------------|-----------|-----------|
| 1,000 | 1,000번 | 10번 |
| 1,000,000 | 1,000,000번 | 20번 |
| 1,000,000,000 | 1,000,000,000번 | 30번 |

---

### 기본 구현

```python
def binary_search(arr, target):
    left, right = 0, len(arr) - 1

    while left <= right:
        mid = (left + right) // 2

        if arr[mid] == target:
            return mid          # 찾으면 인덱스 반환
        elif arr[mid] < target:
            left = mid + 1      # 목표가 오른쪽에 있음
        else:
            right = mid - 1     # 목표가 왼쪽에 있음

    return -1  # 없으면 -1
```

**주의사항**

- `left <= right` 조건: `<` 가 아니라 `<=` 를 써야 원소가 하나 남았을 때도 확인한다.
- `mid = (left + right) // 2`: C/Java 같은 언어에서 오버플로를 피하려면 `left + (right - left) // 2` 로 쓰는 게 안전하다.
- 입력 배열이 **반드시 정렬**돼 있어야 한다.

---

### lower_bound / upper_bound

정확히 일치하는 값이 아니라 "특정 값 이상/초과인 첫 번째 위치"가 필요할 때 활용한다.

#### lower_bound — target 이상인 첫 번째 인덱스

```python
def lower_bound(arr, target):
    left, right = 0, len(arr)  # right = len(arr) (범위 밖)

    while left < right:
        mid = (left + right) // 2
        if arr[mid] < target:
            left = mid + 1
        else:
            right = mid

    return left
```

#### upper_bound — target 초과인 첫 번째 인덱스

```python
def upper_bound(arr, target):
    left, right = 0, len(arr)

    while left < right:
        mid = (left + right) // 2
        if arr[mid] <= target:
            left = mid + 1
        else:
            right = mid

    return left
```

**활용 예시: 배열에서 특정 값의 개수 세기**

```python
arr = [1, 2, 2, 2, 3, 4, 5]

count = upper_bound(arr, 2) - lower_bound(arr, 2)  # 3 - 1 = 2 → 정답: 3개
```

Python에서는 `bisect` 모듈이 이 두 함수를 `bisect_left`, `bisect_right` 로 제공한다.

```python
from bisect import bisect_left, bisect_right

arr = [1, 2, 2, 2, 3, 4, 5]
count = bisect_right(arr, 2) - bisect_left(arr, 2)  # 3
```

---

### 매개변수 탐색 (Parametric Search)

이분 탐색의 가장 강력한 응용이다. **"어떤 조건을 만족하는 최솟값/최댓값"** 을 구하는 문제를 이분 탐색으로 변환한다.

핵심 아이디어: 조건 함수 `f(x)` 가 단조(monotone)하면, 즉 어떤 기준 이상/이하에서 항상 True/False라면 그 경계를 이분 탐색으로 찾는다.

```
f(x) = False False False ... True True True
                               ^
                           이 경계를 찾는다
```

#### 예시: 나무 자르기 (BOJ 2805)

높이 H로 설정된 절단기로 나무를 자를 때, M 미터 이상 얻을 수 있는 최대 높이 H를 구하라.

```python
def can_get(trees, height, m):
    total = sum(max(0, t - height) for t in trees)
    return total >= m

def solve(trees, m):
    left, right = 0, max(trees)

    while left < right:
        mid = (left + right + 1) // 2  # 최댓값을 구할 때는 올림
        if can_get(trees, mid, m):
            left = mid
        else:
            right = mid - 1

    return left
```

**최솟값을 구할 때 vs 최댓값을 구할 때**

| 목적 | mid 계산 | 조건 만족 시 |
|------|----------|-------------|
| 조건 만족하는 **최솟값** | `(left + right) // 2` | `right = mid` |
| 조건 만족하는 **최댓값** | `(left + right + 1) // 2` | `left = mid` |

최댓값을 구할 때 `+ 1` 을 빠뜨리면 `left == right - 1` 상태에서 `mid = left` 가 돼 무한 루프에 빠진다.

---

### 이분 탐색 적용 가능 조건 체크리스트

1. 탐색 공간이 **정렬**돼 있거나 단조 조건을 만족하는가?
2. 탐색 범위(left, right)를 명확히 정의할 수 있는가?
3. 조건 함수를 O(n) 이하로 구현할 수 있는가? → 전체 O(n log n)

---

### 실수하기 쉬운 패턴

```python
# 흔한 실수 1: right = len(arr) - 1 로 해야 할 때 len(arr) 로 잡기
# → 기본 탐색에서는 len(arr) - 1, lower/upper_bound 에서는 len(arr)

# 흔한 실수 2: 무한 루프
# left < right 조건 + 최댓값 탐색에서 mid = (left+right)//2 그대로 쓰기
# → left = mid 인데 mid가 left와 같으면 루프가 안 줄어든다

# 흔한 실수 3: 정수 범위 vs 실수 범위
# 실수 범위 이분 탐색은 반복 횟수 고정 (보통 100회)
left, right = 0.0, 1e9
for _ in range(100):
    mid = (left + right) / 2
    if condition(mid):
        left = mid
    else:
        right = mid
```

---

### 시간 복잡도 정리

| 연산 | 시간 복잡도 |
|------|------------|
| 기본 이분 탐색 | O(log n) |
| lower_bound / upper_bound | O(log n) |
| 매개변수 탐색 (조건 함수 O(n)) | O(n log n) |

---

### 관련 문제 유형

- 정렬된 배열에서 값 찾기, 특정 값의 개수 세기
- "가능한 최솟값/최댓값"을 구하는 최적화 문제
- 이진 탐색 트리(BST)에서의 탐색 (개념 동일)
- 실수 범위 이분 탐색 (소수점 정밀도 문제)
