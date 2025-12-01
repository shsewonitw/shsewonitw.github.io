---
layout: post
title: "[Daily morning study] 동적 계획법 (Dynamic Programming)의 기본 개념"
description: >
  #daily morning study
category: 
    - dms
    - -dsa
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---
# 동적 계획법 (Dynamic Programming)의 기본 개념

동적 계획법(Dynamic Programming, DP)은 문제를 해결하는 효율적인 방법 중 하나로, 복잡한 문제를 보다 간단한 하위 문제로 나누어 해결하는 기법입니다. 이 기법은 특히 최적화 문제에서 많이 사용되며, 중복된 계산을 피하여 성능을 향상시킵니다. 

## 1. 동적 계획법의 기본 원리

동적 계획법은 두 가지 주요 원리를 기반으로 합니다:

1. **최적 부분 구조**: 복잡한 문제의 최적 해는 그 문제의 하위 문제의 최적 해로 구성될 수 있습니다.
2. **중복되는 하위 문제**: 같은 하위 문제가 여러 번 계산될 때, 이전 계산 결과를 재사용하여 성능을 향상시킵니다.

이 두 가지 원리를 활용하여 문제를 해결하는 것이 동적 계획법의 핵심입니다.

## 2. 동적 계획법의 단계

동적 계획법을 적용하는 일반적인 단계는 다음과 같습니다:

1. **문제 정의**: 해결하고자 하는 문제를 명확히 정의합니다.
2. **재귀 관계 수립**: 하위 문제와 그 값을 관계짓는 재귀적 관계를 수립합니다.
3. **기저 사례 정의**: 하위 문제의 기본적인 해를 정의합니다.
4. **메모이제이션 또는 테이블 구축**: 중복 계산을 피하기 위해 하위 문제의 해를 저장하는 방법을 사용합니다.
5. **해결**: 최종 해를 도출합니다.

## 3. 예제: 피보나치 수열

동적 계획법을 설명하기 위해 피보나치 수열을 예로 들어봅시다. 피보나치 수열은 다음과 같이 정의됩니다.

- F(0) = 0
- F(1) = 1
- F(n) = F(n-1) + F(n-2) (for n >= 2)

### 3.1. 재귀적 구현

간단한 재귀적 구현은 아래와 같습니다:

```python
def fibonacci(n):
    if n == 0:
        return 0
    elif n == 1:
        return 1
    else:
        return fibonacci(n-1) + fibonacci(n-2)
```

이 방법은 비효율적입니다. 같은 값이 반복적으로 계산되기 때문입니다.

### 3.2. 메모이제이션 활용

메모리 저장을 통해 중복 계산을 피할 수 있습니다. 이 방법을 사용하면 성능이 크게 향상됩니다.

```python
def fibonacci_memo(n, memo={}):
    if n in memo:
        return memo[n]
    if n == 0:
        return 0
    elif n == 1:
        return 1
    else:
        memo[n] = fibonacci_memo(n-1, memo) + fibonacci_memo(n-2, memo)
        return memo[n]
```

### 3.3. 테이블 기반 접근

또 다른 방법으로는 테이블을 구축하여 하위 문제들을 해결하는 것입니다.

```python
def fibonacci_table(n):
    if n == 0:
        return 0
    elif n == 1:
        return 1
    
    dp = [0] * (n + 1)
    dp[0], dp[1] = 0, 1
    
    for i in range(2, n + 1):
        dp[i] = dp[i - 1] + dp[i - 2]
        
    return dp[n]
```

## 4. 동적 계획법의 응용

동적 계획법은 다양한 문제에 적용할 수 있습니다:

- **최장 공통 부분 수열 (Longest Common Subsequence)**
- **최소 편집 거리 (Edit Distance)**
- **배낭 문제 (Knapsack Problem)**
- **행렬 곱셈 순서 (Matrix Chain Multiplication)**

각 문제는 고유의 재귀 관계를 갖고 있으며, DP를 통해 해결할 수 있습니다.

## 5. 결론

동적 계획법은 복잡한 문제를 보다 간단한 형태로 분할하고, 중복 계산을 피함으로써 효율적으로 해결할 수 있는 강력한 기법입니다. 여러 최적화 문제에 활용되며, 특히 재귀 관계를 잘 정의하고 메모리를 효율적으로 사용하면 성능을 크게 향상시킬 수 있습니다. 동적 계획법의 기본 개념과 다양한 기법을 이해하고 활용하여 문제 해결에 응용할 수 있습니다.
