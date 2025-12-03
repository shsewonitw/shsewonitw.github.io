---
layout: post
title: "[Daily morning study] 해시 테이블(Hash Table)의 작동 원리와 충돌(Collision) 해결 방법"
description: >
  #daily morning study
category: 
    - dms
    - -dsa
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---
# 해시 테이블(Hash Table)의 작동 원리와 충돌(Collision) 해결 방법

해시 테이블은 데이터 구조 중 하나로, 데이터를 키-값 쌍으로 저장하는 방식입니다. 해시 함수(hash function)를 사용하여 키를 해시 값으로 변환하고, 이 해시 값을 사용하여 데이터를 저장하고 검색합니다. 이 가이드는 해시 테이블의 작동 원리와 충돌 해결 방법에 대해 정리합니다.

## 1. 해시 테이블의 기본 개념

### 1.1 해시 함수
해시 함수는 입력된 키를 고정된 크기의 해시 값으로 변환하는 함수입니다. 일반적으로 잘 설계된 해시 함수는 다음과 같은 특성을 가져야 합니다:

- **결정적**: 동일한 입력에 대해 항상 동일한 출력을 생성해야 합니다.
- **충돌 회피**: 서로 다른 두 입력이 같은 해시 값을 생성하는 것을 최소화해야 합니다.
- **빠른 계산**: 해시 값을 신속하게 계산할 수 있어야 합니다.

### 1.2 해시 테이블 구조
해시 테이블은 일반적으로 배열의 형태로 구현됩니다. 배열의 인덱스는 해시 함수를 통해 생성된 해시 값을 기반으로 결정됩니다. 예를 들어, 다음과 같은 구조로 해시 테이블을 설계할 수 있습니다:

```python
class HashTable:
    def __init__(self, size):
        self.size = size
        self.table = [None] * size
```

## 2. 해시 테이블의 동작

### 2.1 삽입(Insert)
삽입 과정은 다음과 같습니다:

1. 키를 해시 함수에 전달하여 해시 값을 계산합니다.
2. 해시 값을 사용해 테이블의 인덱스를 찾습니다.
3. 해당 인덱스에 값을 저장합니다.

예제 코드:

```python
def insert(self, key, value):
    index = hash_function(key) % self.size
    self.table[index] = value  # 충돌 처리가 필요할 수 있음
```

### 2.2 조회(Search)
조회 과정은 다음과 같습니다:

1. 키를 해시 함수에 전달하여 해시 값을 계산합니다.
2. 해시 값을 사용해 테이블의 인덱스를 찾습니다.
3. 해당 인덱스에서 값을 반환합니다.

예제 코드:

```python
def search(self, key):
    index = hash_function(key) % self.size
    return self.table[index]
```

## 3. 충돌(Collision) 해결 방법

충돌이란 서로 다른 두 키가 동일한 해시 값을 생성하는 경우를 말합니다. 이를 해결하는 방법에는 두 가지 주요 방식이 있습니다.

### 3.1 체이닝(Chaining)
체이닝 방법은 각 해시 테이블의 인덱스에 리스트를 두어, 충돌이 발생하면 해당 리스트에 값을 추가하는 방식입니다.

```python
class HashTable:
    def __init__(self, size):
        self.size = size
        self.table = [[] for _ in range(size)]  # 리스트 초기화

    def insert(self, key, value):
        index = hash_function(key) % self.size
        self.table[index].append((key, value))  # 리스트에 추가

    def search(self, key):
        index = hash_function(key) % self.size
        for k, v in self.table[index]:
            if k == key:
                return v
        return None
```

### 3.2 오픈 주소법(Open Addressing)
오픈 주소법은 충돌이 발생했을 때, 다른 인덱스를 찾아 값을 저장하는 방식입니다. 주로 선형 탐사(Linear Probing), 이차 탐사(Quadratic Probing), 더블 해싱(Double Hashing) 등 다양한 기법이 있습니다.

예시로 선형 탐사를 사용하는 경우:

```python
class HashTable:
    def __init__(self, size):
        self.size = size
        self.table = [None] * size

    def insert(self, key, value):
        index = hash_function(key) % self.size
        while self.table[index] is not None:  # 비어있는 위치 찾기
            index = (index + 1) % self.size
        self.table[index] = (key, value)  # 값 저장

    def search(self, key):
        index = hash_function(key) % self.size
        while self.table[index] is not None:
            if self.table[index][0] == key:
                return self.table[index][1]
            index = (index + 1) % self.size
        return None
```

## 4. 해시 테이블의 장단점

### 4.1 장점
- **빠른 데이터 접근**: 평균적으로 O(1)의 시간 복잡도로 데이터 접근이 가능.
- **유연한 데이터 저장**: 키-값 쌍으로 다양한 형식의 데이터를 저장할 수 있음.

### 4.2 단점
- **충돌 처리 필요**: 충돌이 발생할 경우 추가적인 처리 로직이 필요.
- **메모리 낭비**: 배열의 크기가 고정되기 때문에, 더 많은 데이터를 저장할 경우 메모리의 낭비가 있을 수 있음.

## 5. 결론
해시 테이블은 효율적으로 데이터를 저장하고 검색하는 강력한 데이터 구조로, 잘 설계된 해시 함수와 충돌 해결 방법은 성능을 극대화할 수 있습니다. 체이닝과 오픈 주소법은 각기 다른 상황에서 효과적으로 사용할 수 있는 충돌 해결 전략입니다.
