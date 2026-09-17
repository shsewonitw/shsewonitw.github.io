---
layout: post
title: "[Daily morning study] 비트 마스킹(Bit Masking) 기법과 비트 연산"
description: >
  #daily morning study
category: 
    - dms
    - dms-dsa
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 비트 연산 기초

비트 연산은 정수를 2진수로 표현한 상태에서 각 비트 단위로 직접 조작하는 연산이다. 일반 산술 연산보다 훨씬 빠르고, 집합 표현이나 상태 관리에서 메모리 효율이 높다.

### 기본 연산자

| 연산자 | 기호 | 동작 |
|--------|------|------|
| AND | `&` | 두 비트 모두 1일 때만 1 |
| OR | `\|` | 두 비트 중 하나라도 1이면 1 |
| XOR | `^` | 두 비트가 다를 때 1 |
| NOT | `~` | 비트 반전 |
| 왼쪽 시프트 | `<<` | n비트만큼 왼쪽 이동 (× 2^n) |
| 오른쪽 시프트 | `>>` | n비트만큼 오른쪽 이동 (÷ 2^n) |

```python
a = 0b1010  # 10
b = 0b1100  # 12

print(a & b)   # 1000 = 8
print(a | b)   # 1110 = 14
print(a ^ b)   # 0110 = 6
print(~a)      # -11 (부호 있는 정수에서)
print(a << 1)  # 10100 = 20
print(a >> 1)  # 0101 = 5
```

---

## 비트 마스킹이란?

특정 비트 위치를 조작하기 위해 **마스크(mask)**라는 패턴을 만들어 AND, OR, XOR 연산을 적용하는 기법이다. 주로 `1 << k` 형태로 k번째 비트를 가리키는 마스크를 만든다.

### 핵심 연산 패턴

```python
# k번째 비트 확인 (0-indexed)
def check_bit(n, k):
    return (n >> k) & 1  # 0이면 꺼짐, 1이면 켜짐

# k번째 비트 켜기 (set)
def set_bit(n, k):
    return n | (1 << k)

# k번째 비트 끄기 (clear)
def clear_bit(n, k):
    return n & ~(1 << k)

# k번째 비트 토글 (toggle)
def toggle_bit(n, k):
    return n ^ (1 << k)

# 최하위 켜진 비트만 남기기
def lowest_set_bit(n):
    return n & (-n)

# 최하위 켜진 비트 끄기
def clear_lowest_set_bit(n):
    return n & (n - 1)
```

`n & (n-1)` 패턴은 자주 쓰인다. `n`이 0이 될 때까지 반복하면 켜진 비트의 개수를 셀 수 있고, 한 번 적용으로 최하위 1비트를 제거하기 때문에 2의 거듭제곱 판별에도 사용된다.

```python
# n이 2의 거듭제곱인지 확인
def is_power_of_two(n):
    return n > 0 and (n & (n - 1)) == 0

# 켜진 비트 수 세기 (popcount)
def count_bits(n):
    count = 0
    while n:
        n &= (n - 1)
        count += 1
    return count
```

---

## 집합 표현과 부분 집합 열거

비트 마스킹의 가장 강력한 활용은 **집합을 정수 하나로 표현**하는 것이다. n개의 원소가 있을 때 각 원소의 포함 여부를 비트 하나에 대응시키면 2^n가지 부분 집합을 0부터 2^n-1 사이의 정수로 나타낼 수 있다.

```
원소 집합: {A, B, C, D}  →  인덱스: 0, 1, 2, 3
부분 집합 {A, C} → 0101 = 5
부분 집합 {B, D} → 1010 = 10
전체 집합 {A,B,C,D} → 1111 = 15
공집합 → 0000 = 0
```

### 모든 부분 집합 열거

```python
n = 4
elements = ['A', 'B', 'C', 'D']

for mask in range(1 << n):  # 0부터 2^n - 1
    subset = []
    for i in range(n):
        if mask & (1 << i):
            subset.append(elements[i])
    print(f"mask={mask:04b}: {subset}")
```

시간 복잡도: O(2^n × n). 원소가 20개 이하면 충분히 빠르다.

### 특정 집합의 모든 부분 집합 열거

```python
# 집합 S의 모든 부분 집합 (공집합 제외)
def enumerate_subsets(S):
    sub = S
    while sub > 0:
        print(bin(sub))
        sub = (sub - 1) & S  # 핵심 트릭
```

`(sub - 1) & S` 패턴은 S의 부분 집합만 순회한다. sub에서 1을 빼면 최하위 1비트 아래의 비트들이 모두 1로 바뀌는데, S와 AND하면 S에 속하지 않는 비트는 자동으로 걸러진다.

---

## 비트 DP (Bitmask DP)

비트 마스킹의 핵심 응용 분야다. 상태를 비트마스크로 표현해서 방문한 집합이나 선택한 항목을 효율적으로 추적한다.

### TSP (외판원 문제)

n개의 도시를 모두 방문하고 시작점으로 돌아오는 최단 경로 문제.

- `dp[mask][v]`: 집합 `mask`에 있는 도시들을 방문하고 현재 도시 `v`에 있을 때의 최소 비용

```python
import sys
INF = sys.maxsize

def tsp(dist, n):
    # dp[mask][v] = mask 집합을 방문하고 v에 있을 때 최소 비용
    dp = [[INF] * n for _ in range(1 << n)]
    dp[1][0] = 0  # 도시 0에서 시작, 방문 집합 = {0}

    for mask in range(1 << n):
        for v in range(n):
            if dp[mask][v] == INF:
                continue
            if not (mask >> v & 1):
                continue
            # 아직 방문하지 않은 도시 u로 이동
            for u in range(n):
                if mask >> u & 1:
                    continue
                next_mask = mask | (1 << u)
                cost = dp[mask][v] + dist[v][u]
                if cost < dp[next_mask][u]:
                    dp[next_mask][u] = cost

    # 모든 도시 방문 후 0으로 귀환
    full = (1 << n) - 1
    return min(dp[full][v] + dist[v][0] for v in range(1, n))
```

시간 복잡도: O(2^n × n²), 공간 복잡도: O(2^n × n). n이 20 이하에서 실용적이다.

---

## 자주 쓰이는 비트 트릭 정리

```python
# XOR의 성질 활용: 같은 수를 짝수 번 XOR하면 0
# 배열에서 혼자인 원소 찾기
def find_single(nums):
    result = 0
    for n in nums:
        result ^= n
    return result  # 짝수 번 등장한 원소들은 0이 됨

# 두 변수 swap (임시 변수 없이)
def xor_swap(a, b):
    a ^= b
    b ^= a
    a ^= b
    return a, b

# 절대값 (분기 없이)
def abs_no_branch(n):
    mask = n >> 31  # 양수면 0...0, 음수면 1...1
    return (n ^ mask) - mask

# 부호 판별
def sign(n):
    return 1 - (((n >> 31) & 1) << 1)  # 양수: 1, 음수: -1
```

---

## 비트 연산의 한계와 주의점

- **가독성**: 비트 연산은 직관적이지 않아 코드 리뷰나 유지보수가 어렵다. 명확한 주석이 필요하다.
- **부호 있는 정수**: `>>` 연산은 언어마다 다르다. Java에서 `>>>` (논리적 오른쪽 시프트)와 `>>` (산술 오른쪽 시프트)를 구분해야 한다. Python은 정수 크기 제한이 없어 `~n = -(n+1)`이 된다.
- **오버플로우**: C/C++에서 `1 << 31`은 undefined behavior. `1LL << 31` 처럼 64비트 리터럴을 쓰거나 주의가 필요하다.
- **원소 수 제한**: 비트 마스킹 DP는 원소 수가 20~25개를 초과하면 메모리와 시간이 급격히 증가한다.

---

## 정리

비트 마스킹은 집합 연산, 상태 압축, DP 최적화에서 핵심 도구다. `1 << k`로 k번째 비트 마스크 생성, `n & (n-1)`로 최하위 비트 제거, `(sub-1) & S`로 부분 집합 순회하는 세 가지 패턴만 확실히 익혀두면 대부분의 문제에 적용할 수 있다.
