---
layout: post
title: "[Daily morning study] 파이썬 제너레이터(Generator)와 yield"
description: >
  #daily morning study
category: 
    - devlog
    - study
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---
---
title: "파이썬 제너레이터(Generator)와 yield 정리"
layout: post
---

# 파이썬 제너레이터(Generator)와 yield — 학습 가이드

목표: 제너레이터의 개념과 동작 방식을 이해하고, 실제 코드에서 효율적으로 사용하는 방법을 익혀서 다시 볼 수 있도록 정리한 문서다. 예제 코드와 주의점, 연습 문제도 포함되어 있으니 그대로 따라해 보면 이해가 빠르다.

---

## 1. 핵심 개념 요약

- 제너레이터는 이터레이터를 만드는 간단한 방법이다.
- 함수 안에서 `yield`를 사용하면 그 함수는 제너레이터를 반환한다.
- 제너레이터는 값을 하나씩 계산해서 필요할 때마다(요청할 때마다) 반환한다(지연 계산, lazy evaluation).
- 메모리 효율이 좋다(특히 큰 데이터나 무한 시퀀스에 유리).
- 한 번 소비하면 소진된다(재사용하려면 다시 생성해야 함).

---

## 2. 기본 사용법

간단한 예부터 보자.

```python
def count_up_to(n):
    i = 1
    while i <= n:
        yield i
        i += 1

g = count_up_to(3)
print(next(g))  # 1
print(next(g))  # 2
print(next(g))  # 3
# next(g) -> StopIteration
```

`yield`는 값을 반환하고 함수의 상태(로컬 변수, 실행 위치)를 보존한다. 다음 `next()` 호출에서 이어서 실행한다.

for 루프에서는 자동으로 `StopIteration`을 처리해준다.

```python
for x in count_up_to(3):
    print(x)
```

---

## 3. 리스트와의 차이(메모리, 속성)

리스트는 전체를 메모리에 올린다. 제너레이터는 한 번에 하나씩 생성한다.

```python
# 리스트: 메모리에 전체를 저장
nums = [i*i for i in range(10**7)]  # 메모리 많이 사용

# 제너레이터: 필요할 때만 계산
nums_gen = (i*i for i in range(10**7))
```

- 인덱싱, len, 역순 등은 불가능하다(혹은 불편).
- 한 번 순회하면 소진된다.

---

## 4. 제너레이터 표현식(generator expression)

리스트 컴프리헨션과 유사한 문법으로 제너레이터를 만들 수 있다.

```python
gen = (x*x for x in range(5))
print(next(gen))  # 0
```

괄호를 사용한다. 단일 식만 허용.

---

## 5. 실용 예제들

1) 큰 파일을 청크 단위로 읽기

```python
def read_in_chunks(file_path, chunk_size=1024):
    with open(file_path, 'rb') as f:
        while True:
            chunk = f.read(chunk_size)
            if not chunk:
                break
            yield chunk
```

2) 무한 시퀀스(피보나치)

```python
def fibonacci():
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b
```

3) 파이프라인 스타일 처리

```python
def ints(iterable):
    for s in iterable:
        yield int(s)

def even_only(it):
    for x in it:
        if x % 2 == 0:
            yield x

data = ["1", "2", "3", "4"]
pipeline = even_only(ints(data))
print(list(pipeline))  # [2, 4]
```

4) 제너레이터으로 Sieve(간단한 소수 생성기) — 교육용

```python
def odd_iter():
    n = 1
    while True:
        n += 2
        yield n

def not_divisible(n):
    return lambda x: x % n != 0

def primes():
    yield 2
    it = odd_iter()
    while True:
        n = next(it)
        yield n
        it = filter(not_divisible(n), it)
```

---

## 6. yield from (위임)

하위 제너레이터로 제어와 값을 위임할 때 사용한다. 코드가 더 간결해진다.

```python
def subgen():
    yield from range(3)

def gen():
    yield "start"
    yield from subgen()
    yield "end"

print(list(gen()))  # ['start', 0, 1, 2, 'end']
```

`yield from`은 하위 제너레이터에서 발생하는 예외, 반환값(PEP 380) 등을 상위로 위임해준다.

---

## 7. 고급: send(), throw(), close() — 코루틴 기초

제너레이터는 값만 내보내는 생산자 역할 외에 외부로부터 데이터를 받을 수도 있다(간단한 코루틴).

```python
def accumulator():
    total = 0
    while True:
        x = yield total  # yield 후 외부에서 send로 값을 받음
        if x is None:
            break
        total += x
    return total
```

사용법:

```python
c = accumulator()
print(next(c))       # 제너레이터 시작, yield total -> 0
print(c.send(10))    # send 값이 x에 들어가고, 다음 yield에서 total 반환 -> 10
print(c.send(5))     # -> 15
try:
    c.send(None)     # 루프에서 None으로 종료하고 return -> StopIteration(value)
except StopIteration as e:
    print("Returned:", e.value)
```

- `send(value)`는 `next()`처럼 동작하지만, 바로 전 `yield` 표현식의 값으로 `value`를 전달한다.
- `throw(exc)`는 제너레이터 내부에서 해당 예외를 발생시킨다.
- `close()`는 `GeneratorExit`를 발생시켜 정리 코드를 실행하게 한다.

주의: 코루틴 스타일로 쓸 때는 처음에 반드시 `next(g)` 또는 `g.send(None)`로 시작해야 한다(첫 send는 yield가 대기하고 있어야 한다).

---

## 8. 제너레이터에서의 반환값

Python 3.3+에서는 `return value`를 사용해 제너레이터를 종료하면서 값을 전달할 수 있다. 이 값은 `StopIteration.value`로 얻는다(주로 `yield from` 패턴에서 유용).

```python
def sub():
    yield 1
    yield 2
    return "done"

def main():
    result = yield from sub()
    print("sub returned:", result)

g = main()
list(g)  # sub returned: done
```

직접 `next()`/`send()`를 사용하는 코드에서 `StopIteration`의 `value`를 잡아 처리할 수도 있다.

---

## 9. 성능/메모리 관련 팁

- 큰 데이터 처리: 제너레이터 사용으로 메모리 절약.
- 작은 데이터/복수 접근: 리스트가 더 빠를 수 있다(인덱싱/랜덤 접근).
- 이터레이터 체이닝/필터링 등 반복 기반 처리에서 제너레이터와 itertools를 같이 쓰면 아주 효율적이다.

예: itertools와 함께

```python
import itertools
nums = (i for i in range(1000000))
filtered = itertools.islice(nums, 100)  # 필요 요소만 취함
```

---

## 10. 흔히 하는 실수와 주의사항

- 제너레이터는 한 번만 순회 가능하다. 다시 사용하려면 재생성해야 한다.
- `len()`이나 인덱스 접근 불가.
- 데버깅 시 내부 상태 확인이 어려울 수 있다(프린트나 로거로 상태 찍어보기).
- `yield`와 `return`의 혼동: `return`은 제너레이터를 끝내고 `StopIteration` 발생. `yield`는 값을 내보내며 함수 상태 보존.
- 파일 핸들링과 같이 컨텍스트 매니저와 같이 사용하는 경우, 제너레이터가 파일을 닫기 전에 소비가 끝나버리면 문제가 생긴다. 예:

```python
def bad_reader(path):
    with open(path) as f:
        for line in f:
            yield line

g = bad_reader("file.txt")
# with 블록이 함수 내부라 괜찮지만, 호출 시점에 소비가 늦으면 파일이 닫힐 수 있음
# 실제로는 위 코드는 제너레이터가 열려있는 동안 파일은 닫히지 않음(함수 scope 유지)
```

더 안전한 패턴은 파일을 제너레이터 외부에서 열고 명시적으로 닫는 것이다.

---

## 11. 디버깅 팁

- 제너레이터 내부에 logging/print를 넣어 상태를 확인하자.
- 작은 예제로 동작을 먼저 확인하자.
- `inspect.getgeneratorstate(g)`로 상태 확인 가능(`GEN_CREATED`, `GEN_RUNNING`, `GEN_SUSPENDED`, `GEN_CLOSED`).

```python
import inspect
g = count_up_to(3)
print(inspect.getgeneratorstate(g))  # GEN_CREATED
next(g)
print(inspect.getgeneratorstate(g))  # GEN_SUSPENDED
```

---

## 12. 연습 문제(스스로 풀어보기)

1. 파일을 라인 단위로 읽되, 특정 키워드가 나올 때까지의 라인만 제너레이터로 반환하는 함수를 만들어라.
2. 무한 소수 생성기를 만들어서 100번째 소수를 출력하라.
3. 두 개의 정렬된 제너레이터를 받아서 병합(merge)하는 제너레이터를 구현하라(병합 정렬의 merge와 유사).

힌트와 해답은 아래에 있다(먼저 직접 풀어본 뒤 확인해라).

힌트:
- 1번은 파일을 한 줄씩 읽어 `yield`하면 된다.
- 2번은 단순 소수 판별(제곱근까지 검사)로 충분하다.
- 3번은 두 제너레이터에서 각각 next를 호출해 현재 값을 비교하며 작은 것을 yield한다.

간단한 해답 예시(참고용):

```python
# 1번
def lines_until(filename, keyword):
    with open(filename, 'r', encoding='utf-8') as f:
        for line in f:
            if keyword in line:
                break
            yield line

# 2번 (단순)
def is_prime(n):
    if n < 2:
        return False
    i = 2
    while i*i <= n:
        if n % i == 0:
            return False
        i += 1
    return True

def prime_gen():
    n = 2
    while True:
        if is_prime(n):
            yield n
        n += 1

g = prime_gen()
for _ in range(99):
    next(g)
print(next(g))  # 100번째 소수

# 3번
def merge(a, b):
    a = iter(a)
    b = iter(b)
    try:
        av = next(a)
    except StopIteration:
        yield from b
        return
    try:
        bv = next(b)
    except StopIteration:
        yield av
        yield from a
        return

    while True:
        if av <= bv:
            yield av
            try:
                av = next(a)
            except StopIteration:
                yield bv
                yield from b
                return
        else:
            yield bv
            try:
                bv = next(b)
            except StopIteration:
                yield av
                yield from a
                return
```

---

## 13. 요약(cheat sheet)

- 제너레이터 만들기: 함수 안에서 `yield` 사용.
- 제너레이터 호출: `g = gen()` — 소비: `next(g)`, `for x in g: ...`, `list(g)`.
- 한 번 소비되면 끝. 재사용하려면 새로 생성.
- `send(value)`: 마지막 `yield`의 표현식 결과로 전달된다. 처음엔 `next(g)` 또는 `g.send(None)` 필요.
- `yield from subgen()`으로 하위 제너레이터 위임.
- `return value` -> `StopIteration.value`(PEP 380).
- 메모리 효율 + 스트리밍 처리에 강함. 복잡한 상태 기계나 비동기/코루틴 전환의 기초로 활용 가능.

---

필요하면 각 예제를 더 자세히 풀어보고, 실제 케이스(로그 처리, 네트워크 스트리밍, ETL 파이프라인)로 확장된 예제를 추가로 정리할게.
