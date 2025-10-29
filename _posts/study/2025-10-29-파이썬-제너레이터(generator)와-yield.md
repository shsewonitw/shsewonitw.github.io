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
title: 파이썬 제너레이터(Generator)와 yield
description: 파이썬 제너레이터의 개념, 문법, 활용 예시(파일 처리, 무한 시퀀스, 파이프라인 등), 고급 기능(yield from, send, close)과 실습 문제 및 해답을 정리한 학습 가이드
---

# 파이썬 제너레이터(Generator)와 yield

이 문서는 파이썬 제너레이터의 핵심 개념과 실무적 활용법을 정리한 학습 가이드야. 코드를 직접 실행하면서 읽어보고, 연습 문제도 풀어보면 이해가 빨리 될 거야.

목표
- 제너레이터와 yield의 동작 원리를 이해한다.
- 제너레이터를 이용해 메모리 효율적인 스트리밍 처리와 파이프라인을 구현할 수 있다.
- yield from, send, close, throw 같은 고급 기능을 실무에 적용할 수 있다.

사전 지식
- 파이썬 기본 문법(함수, 반복문, 예외 처리)
- 이터레이터(iterator)와 이터러블(iterable)의 개념을 알고 있으면 좋다

---

## 1. 기본 개념: 이터레이터와 제너레이터

- 이터러블(iterable): `for` 루프로 순회 가능한 객체. 예: 리스트, 튜플, 문자열, 파일 객체 등.
- 이터레이터(iterator): `__next__()` 메서드를 가진 객체. `next(it)`로 다음 값을 얻을 수 있고, 더 이상 값이 없으면 `StopIteration`을 발생시킨다.
- 제너레이터(generator): 이터레이터를 만드는 간단한 방법. 함수 안에서 `yield` 키워드를 사용하면 그 함수는 제너레이터 함수를 반환한다. 제너레이터 함수가 호출되면 제너레이터 객체(이터레이터)를 만든다.

간단한 예:

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
# 다음 호출은 StopIteration 발생
```

`for x in count_up_to(3):` 처럼 `for`에서 바로 사용할 수 있다.

---

## 2. yield vs return

- `return`은 함수 실행을 끝내면서 값을 반환하고 종료한다.
- `yield`는 값을 반환(생산)하면서 함수의 상태(로컬 변수, 실행 위치)를 보존한다. 이후 `next()`가 호출되면 이어서 실행을 계속한다.
- 제너레이터는 한 번만 순회할 수 있다(단일 소비). 다회 반복하려면 새 제너레이터를 만들어야 한다.

예: `return`과 `yield`의 차이

```python
def f_return():
    return [1, 2, 3]

def f_yield():
    yield 1
    yield 2
    yield 3
```

`f_return()`는 리스트를 한 번에 만들어 메모리를 사용하지만, `f_yield()`는 한 번에 하나씩 생성한다.

---

## 3. 제너레이터 표현식(Generator expression)

리스트 컴프리헨션과 유사하지만 `[]` 대신 `()` 사용:

```python
gen = (x*x for x in range(5))
print(next(gen))  # 0
for v in gen:
    print(v)  # 1, 4, 9, 16
```

제너레이터 표현식은 메모리를 거의 사용하지 않고 지연 평가(lazy evaluation)를 제공한다.

---

## 4. 실용 예제: 대용량 파일 처리

큰 로그 파일을 한 줄씩 처리할 때 제너레이터가 유용하다.

```python
def read_lines(path):
    with open(path, 'r', encoding='utf-8') as f:
        for line in f:
            yield line.rstrip('\n')

def grep(pattern, lines):
    for line in lines:
        if pattern in line:
            yield line

# 사용 예
lines = read_lines('big.log')
matched = grep('ERROR', lines)
for m in matched:
    process(m)  # process는 사용자 정의 처리 함수
```

이렇게 하면 파일 전체를 메모리에 올리거나 중간 리스트를 만들지 않고 파이프라인처럼 처리할 수 있다.

---

## 5. 무한 시퀀스와 지연 생성

제너레이터는 무한 시퀀스를 만들 때도 유용하다.

```python
def naturals(start=1):
    n = start
    while True:
        yield n
        n += 1

# 사용: itertools.islice와 함께
import itertools
for x in itertools.islice(naturals(), 0, 10):
    print(x)
```

주의: 무한 제너레이터는 반드시 소비 시 제한을 걸어야(예: islice) 무한 루프를 피한다.

---

## 6. yield from (PEP 380) — 서브제너레이터 위임

하나의 제너레이터에서 다른 제너레이터의 값을 그대로 전달하고, 서브 제너레이터의 반환값(return value)을 얻을 수 있다.

```python
def gen_inner():
    yield 1
    yield 2
    return 'done'  # Python 3.3+에서 가능

def gen_outer():
    result = yield from gen_inner()
    print('inner returned:', result)
    yield 3

for x in gen_outer():
    print(x)
# 출력:
# 1
# 2
# inner returned: done
# 3
```

`yield from`은 중첩 제너레이터를 깔끔하게 연결해준다. 또한 예외 전파와 close 동작을 자동으로 처리한다.

---

## 7. 고급: send(), throw(), close() — 코루틴 스타일 사용

제너레이터는 단순 생산자뿐 아니라 '코루틴' 형태로 값을 보내고 상호작용할 수 있다.

- send(value): 제너레이터 내부의 `yield` 표현식에 값을 전달한다. 처음 호출은 None을 보내야 한다(제너레이터 시작 전).
- throw(exc): 제너레이터 내부에 예외를 발생시킨다(try/except로 잡을 수 있음).
- close(): 제너레이터에 GeneratorExit를 발생시키고 정리할 기회를 준다.

예: 간단한 코루틴(에코)

```python
def echo():
    try:
        while True:
            received = yield
            print('received', received)
    except GeneratorExit:
        print('echo closed')

c = echo()
next(c)         # 제너레이터를 시작(prime)
c.send('hello') # received hello
c.send('world') # received world
c.close()       # echo closed
```

`yield`가 표현식으로 사용될 때는 값을 받을 수 있다:

```python
def accumulator():
    total = 0
    while True:
        x = yield total  # yield가 값을 반환하고, 외부에서 send로 x를 전달
        if x is None:
            break
        total += x
    return total

c = accumulator()
print(next(c))     # 0 (초기 total)
print(c.send(10))  # 10 (이전 total 반환), 내부 total은 10
print(c.send(5))   # 15
try:
    c.send(None)   # 종료 신호
except StopIteration as e:
    print('final total via StopIteration.value =', e.value)
```

주의: 제너레이터가 반환(return)하면 그 값은 StopIteration의 value 속성으로 전달된다. (Python 3.3+)

---

## 8. 실용적인 파이프라인 예시

로그 스트림을 받아 JSON으로 파싱하고 필터링해서 통계내기:

```python
import json

def read_lines(path):
    with open(path, 'r', encoding='utf-8') as f:
        for line in f:
            yield line

def parse_json(lines):
    for line in lines:
        try:
            yield json.loads(line)
        except json.JSONDecodeError:
            continue

def filter_level(records, level):
    for r in records:
        if r.get('level') == level:
            yield r

def count_by_key(records, key):
    counts = {}
    for r in records:
        k = r.get(key)
        counts[k] = counts.get(k, 0) + 1
    yield counts

# 파이프라인 연결
lines = read_lines('logs.jsonl')
records = parse_json(lines)
errors = filter_level(records, 'ERROR')
counts = next(count_by_key(errors, 'service'))
print(counts)
```

각 단계가 제너레이터라서 중간 결과를 메모리에 올리지 않고 스트림으로 처리할 수 있다.

---

## 9. 성능 및 메모리 고려사항

- 장점
  - 메모리 사용량 감소: 필요한 값만 생성.
  - 명료한 스트리밍 파이프라인 작성 가능.
  - 무한 시퀀스 표현 가능.

- 단점 / 주의사항
  - 제너레이터는 함수 호출/상태 저장 비용이 있으므로 초단기 반복(엄청 많은 작은 연산)에서는 리스트보다 느릴 수 있다.
  - 한 번 소비하면 재사용 불가: 결과를 여러 번 사용하려면 리스트로 변환하거나 제너레이터를 여러 번 생성해야 한다.
  - 디버깅이 까다로울 수 있다(실행 위치 보관).
  - 예외 처리와 반환 값(Pep 380) 동작을 잘 이해해야 한다.

---

## 10. 자주 쓰이는 패턴 모음

- chunked 처리(큰 시퀀스를 청크 단위로 처리)

```python
def chunked(iterable, size):
    it = iter(iterable)
    while True:
        chunk = []
        try:
            for _ in range(size):
                chunk.append(next(it))
        except StopIteration:
            if chunk:
                yield chunk
            break
        yield chunk
```

- sliding window (윈도우 슬라이딩)

```python
from collections import deque

def sliding_window(iterable, n):
    it = iter(iterable)
    window = deque(maxlen=n)
    for _ in range(n):
        try:
            window.append(next(it))
        except StopIteration:
            return
    yield tuple(window)
    for x in it:
        window.append(x)
        yield tuple(window)
```

- flatten (중첩 리스트 평탄화)

```python
def flatten(nested):
    for item in nested:
        if isinstance(item, (list, tuple)):
            for sub in flatten(item):
                yield sub
        else:
            yield item
```

---

## 11. 디버깅 팁

- 제너레이터 내부 상태를 확인하려면 간단한 로그(프린트)를 넣어라.
- 재사용이 필요하면 결과를 리스트로 받아두거나, 제너레이터를 생성하는 함수를 호출하는 편이 안전하다.
- `inspect.getgeneratorstate(gen)`로 제너레이터 상태(NEW, RUNNING, SUSPENDED, CLOSED)를 확인할 수 있다.

```python
import inspect
inspect.getgeneratorstate(gen)
```

---

## 12. 연습 문제(스스로 풀어보기)

1) 파일에서 n줄씩 묶어 리스트로 반환하는 `read_in_chunks(path, n)` 제너레이터를 작성하라. 각 청크는 최대 n줄이다.

2) 자연수에서 소수만 생성하는 무한 제너레이터 `primes()`를 간단한 에라토스테네스 방식(메모리 제한된 간단 버전)으로 작성해라.

3) 제너레이터와 `send()`를 이용해 간단한 누적 합 코루틴을 만들라. 외부에서 값을 보내면 누적하고, 특정 값(예: None)을 보내면 종료하고 최종 합계를 반환하라.

4) 중첩된 이터러블(리스트/튜플)에서 숫자만 yield하는 `deep_numbers()` 제너레이터를 작성하라.

---

## 13. 연습 해답 (모범 답안)

1) read_in_chunks

```python
def read_in_chunks(path, n):
    with open(path, 'r', encoding='utf-8') as f:
        chunk = []
        for line in f:
            chunk.append(line.rstrip('\n'))
            if len(chunk) >= n:
                yield chunk
                chunk = []
        if chunk:
            yield chunk
```

2) primes (간단한 생성기 — 희소한 검사)

```python
def primes():
    D = {}  # 합성수의 최소 소인수 기록: composite -> [prime1, prime2, ...]
    q = 2
    while True:
        if q not in D:
            # q는 소수
            yield q
            D[q*q] = [q]
        else:
            for p in D[q]:
                D.setdefault(p+q, []).append(p)
            del D[q]
        q += 1
```

(위 코드는 일반적인 "incremental sieve" 아이디어의 한 예야. 메모리/성능 면에서 여러 개선이 가능해.)

3) 누적 합 코루틴

```python
def accumulator():
    total = 0
    try:
        while True:
            x = yield total
            if x is None:
                return total
            total += x
    except GeneratorExit:
        return total

# 사용 예
c = accumulator()
print(next(c))      # 0
print(c.send(10))   # 10
print(c.send(5))    # 15
try:
    c.send(None)
except StopIteration as e:
    print('final:', e.value)  # final: 15
```

4) deep_numbers

```python
def deep_numbers(obj):
    if isinstance(obj, (list, tuple)):
        for item in obj:
            yield from deep_numbers(item)
    elif isinstance(obj, (int, float)):
        yield obj
    else:
        # 무시하거나 필요시 변환 추가
        return
```

---

## 14. 정리(핵심 요점)

- 제너레이터는 함수 상태를 유지하면서 값을 하나씩 생성하는 이터레이터다.
- 메모리 효율, 지연 평가, 파이프라인 구성에 유리하다.
- `yield from`으로 서브제너레이터를 위임하고 반환값을 받을 수 있다.
- `send`, `throw`, `close`로 제너레이터와 양방향 통신(코루틴 스타일)이 가능하다.
- 제너레이터는 한 번만 소비될 수 있으므로 재사용이 필요하면 새로 생성하거나 결과를 저장하라.

---

참고 자료
- 파이썬 공식 문서: https://docs.python.org/3/reference/expressions.html#yieldexpr
- PEP 342, PEP 380 (send, yield from 관련)
- itertools 모듈 문서

필요하면 각 예제를 더 확장하거나 실무 예제로 바꿔서 정리해줄게.
