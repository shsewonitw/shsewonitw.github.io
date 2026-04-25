---
layout: post
title: "[Daily morning study] 정렬 알고리즘 비교 (Quick Sort, Merge Sort, Heap Sort)"
description: >
  #daily morning study
category: 
    - dms
    - dms-dsa
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 정렬 알고리즘 비교

실무에서 직접 정렬을 구현하는 경우는 드물지만, 각 알고리즘의 원리와 시간 복잡도를 이해하면 내장 정렬 함수의 동작을 예측하고 성능 문제를 진단하는 데 도움이 된다.

---

## Quick Sort (퀵 정렬)

### 원리

피벗(pivot) 원소를 하나 선택한 뒤, 피벗보다 작은 원소는 왼쪽, 큰 원소는 오른쪽으로 분할하는 과정을 재귀적으로 반복한다. 분할 정복(Divide and Conquer) 전략이다.

```
[3, 6, 8, 10, 1, 2, 1]  → pivot = 3
 left: [1, 2, 1]   pivot: [3]   right: [6, 8, 10]
 → 각 부분을 재귀적으로 정렬
```

### 구현 (Python)

```python
def quick_sort(arr):
    if len(arr) <= 1:
        return arr
    pivot = arr[len(arr) // 2]
    left  = [x for x in arr if x < pivot]
    mid   = [x for x in arr if x == pivot]
    right = [x for x in arr if x > pivot]
    return quick_sort(left) + mid + quick_sort(right)
```

### 시간 복잡도

| 경우 | 복잡도 |
|------|--------|
| 평균 | O(n log n) |
| 최악 (이미 정렬된 배열 + 고정 피벗) | O(n²) |
| 공간 복잡도 | O(log n) ~ O(n) (재귀 스택) |

### 특징

- 제자리 정렬(in-place)이 가능해 메모리 효율이 좋다.
- 캐시 지역성(cache locality)이 좋아 실제 성능이 빠른 편이다.
- 피벗 선택이 성능의 핵심이다. 랜덤 피벗이나 Median-of-three 전략으로 최악 케이스를 피한다.
- 불안정 정렬(unstable sort)이다.

---

## Merge Sort (병합 정렬)

### 원리

배열을 절반으로 계속 분할한 뒤, 정렬된 두 부분 배열을 합치는(merge) 방식이다. 항상 O(n log n)을 보장한다.

```
[38, 27, 43, 3]
→ [38, 27] | [43, 3]
→ [38] [27] | [43] [3]
← [27, 38]  ← [3, 43]
← [3, 27, 38, 43]
```

### 구현 (Python)

```python
def merge_sort(arr):
    if len(arr) <= 1:
        return arr
    mid = len(arr) // 2
    left  = merge_sort(arr[:mid])
    right = merge_sort(arr[mid:])
    return merge(left, right)

def merge(left, right):
    result = []
    i = j = 0
    while i < len(left) and j < len(right):
        if left[i] <= right[j]:
            result.append(left[i]); i += 1
        else:
            result.append(right[j]); j += 1
    return result + left[i:] + right[j:]
```

### 시간 복잡도

| 경우 | 복잡도 |
|------|--------|
| 최선 / 평균 / 최악 | O(n log n) |
| 공간 복잡도 | O(n) (임시 배열 필요) |

### 특징

- 안정 정렬(stable sort)이다. 동일한 값의 원소 순서가 유지된다.
- 추가 메모리(O(n))가 필요하다. 메모리가 제한적인 환경에서는 단점이 된다.
- 연결 리스트 정렬에 특히 유리하다 (포인터 연결만 바꾸면 돼서 추가 메모리 없이 가능).
- Java의 `Arrays.sort(Object[])` (TimSort 기반)가 Merge Sort를 활용한다.

---

## Heap Sort (힙 정렬)

### 원리

배열을 최대 힙(Max-Heap)으로 구성한 뒤, 루트(최댓값)를 배열 끝으로 이동시키고 힙 크기를 줄이며 반복한다.

```
1단계: 배열 → Max-Heap 구성
2단계: 루트(최댓값)와 마지막 원소 교환 → 힙 크기 -1
3단계: Heapify(루트 복원) → 반복
```

### 구현 (Python)

```python
def heapify(arr, n, i):
    largest = i
    left, right = 2 * i + 1, 2 * i + 2
    if left < n and arr[left] > arr[largest]:
        largest = left
    if right < n and arr[right] > arr[largest]:
        largest = right
    if largest != i:
        arr[i], arr[largest] = arr[largest], arr[i]
        heapify(arr, n, largest)

def heap_sort(arr):
    n = len(arr)
    for i in range(n // 2 - 1, -1, -1):  # 최대 힙 구성
        heapify(arr, n, i)
    for i in range(n - 1, 0, -1):         # 정렬
        arr[0], arr[i] = arr[i], arr[0]
        heapify(arr, i, 0)
    return arr
```

### 시간 복잡도

| 경우 | 복잡도 |
|------|--------|
| 최선 / 평균 / 최악 | O(n log n) |
| 공간 복잡도 | O(1) (제자리 정렬) |

### 특징

- 최악의 경우에도 O(n log n)을 보장하면서 추가 메모리가 O(1)이다.
- 불안정 정렬이다.
- 캐시 지역성이 나빠서 Quick Sort보다 실제 성능이 느린 경우가 많다.
- 메모리가 극도로 제한된 임베디드 환경에서 유용하다.

---

## 세 알고리즘 비교 요약

| 알고리즘 | 평균 | 최악 | 공간 | 안정성 |
|----------|------|------|------|--------|
| Quick Sort | O(n log n) | O(n²) | O(log n) | 불안정 |
| Merge Sort | O(n log n) | O(n log n) | O(n) | 안정 |
| Heap Sort | O(n log n) | O(n log n) | O(1) | 불안정 |

---

## 실무에서 어떤 게 쓰이나?

- **Python `sorted()` / `list.sort()`** — TimSort 사용 (Merge Sort + Insertion Sort 혼합). 안정 정렬.
- **Java `Arrays.sort(int[])`** — Dual-Pivot Quick Sort 사용.
- **Java `Arrays.sort(Object[])`** — TimSort 사용 (안정 정렬 보장).
- **C++ `std::sort`** — IntroSort 사용 (Quick Sort + Heap Sort + Insertion Sort 혼합). 최악 O(n log n) 보장.

대부분의 언어 표준 라이브러리는 순수 Quick Sort나 Merge Sort를 단독으로 쓰지 않는다. 각각의 단점을 보완한 하이브리드 알고리즘을 채택한다.

---

## 핵심 정리

- Quick Sort: 평균적으로 가장 빠르고 캐시 효율 좋음. 최악 O(n²) 주의.
- Merge Sort: 항상 O(n log n). 안정 정렬 필요할 때 선택. 추가 메모리 필요.
- Heap Sort: 항상 O(n log n)이면서 O(1) 공간. 하지만 캐시 비효율로 실제론 느림.
- 면접에서는 각 알고리즘의 시간 복잡도와 안정성 여부, 언제 어떤 걸 선택할지를 설명할 수 있어야 한다.
