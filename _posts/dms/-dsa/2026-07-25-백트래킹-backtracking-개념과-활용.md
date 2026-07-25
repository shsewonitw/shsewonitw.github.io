---
layout: post
title: "[Daily morning study] 백트래킹 (Backtracking) 개념과 활용"
description: >
  #daily morning study
category: 
    - dms
    - dms-dsa
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 백트래킹이란?

백트래킹은 해를 찾기 위해 가능한 모든 경우를 탐색하되, 해가 될 가능성이 없는 경우를 조기에 포기(가지치기, pruning)하는 탐색 기법이다. DFS를 기반으로 동작하며, 브루트 포스보다 불필요한 탐색을 줄여 효율적이다.

핵심 아이디어:
1. 후보를 선택한다.
2. 선택이 유효한지 검사한다. (제약 조건 확인)
3. 유효하면 재귀적으로 다음 단계로 진행한다.
4. 해가 완성되거나 더 이상 진행할 수 없으면 되돌아간다. (backtrack)

---

## 백트래킹 vs 브루트 포스

| 항목 | 브루트 포스 | 백트래킹 |
|------|------------|---------|
| 탐색 방식 | 모든 경우 완전 탐색 | 가지치기로 불필요한 탐색 제거 |
| 효율성 | 낮음 | 상대적으로 높음 |
| 구현 복잡도 | 낮음 | 중간 |
| 적합한 상황 | 경우의 수가 매우 적을 때 | 제약 조건이 많을 때 |

백트래킹은 최악의 경우 여전히 지수 시간이지만, 실제로는 가지치기 덕분에 훨씬 빠르게 동작한다.

---

## 기본 구조

```python
def backtrack(state, choices):
    if is_solution(state):          # 해를 찾은 경우
        record(state)
        return

    for choice in choices:
        if is_valid(state, choice): # 가지치기
            make(state, choice)     # 선택
            backtrack(state, next_choices(state, choice))
            unmake(state, choice)   # 선택 취소 (복원)
```

`make`와 `unmake`가 짝을 이루는 구조가 핵심이다. 선택을 적용했다가 재귀 호출이 끝나면 반드시 복원해야 다음 후보를 올바르게 탐색할 수 있다.

---

## 예시 1: N-Queens 문제

N×N 체스판에 N개의 퀸을 서로 공격하지 않도록 배치하는 문제.

```python
def solve_n_queens(n):
    result = []
    queens = []  # queens[row] = col

    def is_valid(row, col):
        for r, c in enumerate(queens):
            if c == col:             # 같은 열
                return False
            if abs(r - row) == abs(c - col):  # 같은 대각선
                return False
        return True

    def backtrack(row):
        if row == n:
            result.append(queens[:])
            return
        for col in range(n):
            if is_valid(row, col):
                queens.append(col)
                backtrack(row + 1)
                queens.pop()         # 선택 취소

    backtrack(0)
    return result

print(len(solve_n_queens(8)))  # 92
```

행을 하나씩 내려가면서 각 행에 퀸을 배치한다. `is_valid`로 열, 대각선 충돌을 확인해 가지치기한다.

---

## 예시 2: 부분집합 합 (Subset Sum)

주어진 숫자 집합에서 합이 target이 되는 모든 부분집합을 구하는 문제.

```python
def subset_sum(nums, target):
    result = []
    nums.sort()  # 정렬 후 가지치기 강화

    def backtrack(start, current, current_sum):
        if current_sum == target:
            result.append(current[:])
            return
        for i in range(start, len(nums)):
            if current_sum + nums[i] > target:  # 가지치기
                break
            current.append(nums[i])
            backtrack(i + 1, current, current_sum + nums[i])
            current.pop()

    backtrack(0, [], 0)
    return result

print(subset_sum([2, 3, 6, 7], 7))
# [[7], [3, 7]가 아니라, 실제 답: [[7]]... 직접 실행해서 확인
```

정렬 후 `current_sum + nums[i] > target`이면 이후 숫자는 더 크므로 탐색을 조기 종료한다.

---

## 예시 3: 순열 생성

```python
def permutations(nums):
    result = []
    used = [False] * len(nums)

    def backtrack(current):
        if len(current) == len(nums):
            result.append(current[:])
            return
        for i in range(len(nums)):
            if not used[i]:
                used[i] = True
                current.append(nums[i])
                backtrack(current)
                current.pop()
                used[i] = False

    backtrack([])
    return result

print(permutations([1, 2, 3]))
# [[1,2,3], [1,3,2], [2,1,3], [2,3,1], [3,1,2], [3,2,1]]
```

`used` 배열로 이미 선택한 원소를 추적한다. 재귀 후 `used[i] = False`로 반드시 복원한다.

---

## 예시 4: 스도쿠 풀기

```python
def solve_sudoku(board):
    def is_valid(row, col, num):
        # 행 확인
        if num in board[row]:
            return False
        # 열 확인
        if num in [board[r][col] for r in range(9)]:
            return False
        # 3x3 박스 확인
        box_row, box_col = 3 * (row // 3), 3 * (col // 3)
        for r in range(box_row, box_row + 3):
            for c in range(box_col, box_col + 3):
                if board[r][c] == num:
                    return False
        return True

    def backtrack():
        for row in range(9):
            for col in range(9):
                if board[row][col] == 0:  # 빈 칸
                    for num in range(1, 10):
                        if is_valid(row, col, num):
                            board[row][col] = num
                            if backtrack():
                                return True
                            board[row][col] = 0  # 복원
                    return False  # 어떤 숫자도 안 됨 → backtrack
        return True  # 모든 칸 채움

    backtrack()
```

빈 칸을 만나면 1~9를 시도하고, 유효하지 않으면 0으로 복원(backtrack)한다.

---

## 가지치기 전략

백트래킹의 효율은 가지치기 품질에 달려 있다. 주요 전략:

**1. 제약 전파 (Constraint Propagation)**
- 선택 후 즉시 이후 선택지를 좁혀나간다.
- 스도쿠에서 한 칸에 숫자를 넣으면 같은 행/열/박스의 가능한 숫자를 업데이트하는 식.

**2. 최소 잔여 값 (MRV, Minimum Remaining Values)**
- 가능한 선택지가 가장 적은 변수부터 처리한다.
- 실패를 조기에 탐지할 확률이 높아진다.

**3. 경계 검사 (Bound Checking)**
- 현재 상태에서 목표에 도달할 수 없음을 수학적으로 판단한다.
- 부분집합 합 문제에서 남은 원소 합이 목표에 미달하면 포기.

---

## 시간 복잡도

백트래킹의 시간 복잡도는 문제에 따라 크게 다르다.

| 문제 | 최악 복잡도 |
|------|------------|
| 순열 | O(n!) |
| 부분집합 | O(2ⁿ) |
| N-Queens | O(n!) (실제는 가지치기로 훨씬 빠름) |
| 스도쿠 | O(9^(빈 칸 수)) |

가지치기 효과로 평균 성능은 최악보다 훨씬 좋다. 하지만 최악을 항상 고려해야 한다.

---

## 백트래킹이 잘 맞는 문제 유형

- **조합/순열 탐색**: 가능한 조합 중 조건을 만족하는 것 탐색
- **제약 충족 문제 (CSP)**: N-Queens, 스도쿠, 그래프 색칠 문제
- **경로 탐색**: 미로 탈출, 해밀턴 경로
- **파티셔닝**: 집합을 조건에 맞게 분할하는 문제

DP로 해결 가능한 문제와 혼동하기 쉬운데, DP는 중복 부분 구조가 있고 최적값을 구할 때 쓰고, 백트래킹은 전체 해의 집합을 열거하거나 단일 가능한 해를 찾을 때 쓴다.

---

## 정리

- 백트래킹은 DFS + 가지치기로, 불가능한 경로를 조기에 포기한다.
- `make → recurse → unmake` 패턴을 일관되게 유지해야 상태 복원이 정확하다.
- 가지치기 조건을 얼마나 잘 설계하느냐가 성능의 핵심이다.
- N-Queens, 스도쿠, 순열, 부분집합 합 등이 대표 문제다.
