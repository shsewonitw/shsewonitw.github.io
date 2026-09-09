---
layout: post
title: "[Daily morning study] TypeScript 제네릭(Generics)과 조건부 타입(Conditional Types)"
description: >
  #daily morning study
category: 
    - dms
    - -front
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 제네릭(Generics)이란

제네릭은 타입을 파라미터처럼 다룰 수 있게 해주는 기능이다. 특정 타입에 종속되지 않고 다양한 타입에 동작하는 재사용 가능한 컴포넌트를 만들 수 있다.

```typescript
// 제네릭 없이 작성하면 타입이 고정됨
function identity(arg: number): number {
  return arg;
}

// any를 쓰면 타입 안전성을 잃음
function identity(arg: any): any {
  return arg;
}

// 제네릭을 쓰면 타입 안전성을 유지하면서 유연하게
function identity<T>(arg: T): T {
  return arg;
}

const result = identity<string>("hello"); // result: string
const result2 = identity(42);            // result2: number (타입 추론)
```

`T`는 관습적으로 사용하는 이름이지만, 어떤 식별자든 쓸 수 있다. `K`, `V`, `U` 같은 이름도 흔히 쓴다.

---

## 제네릭 함수

```typescript
// 배열에서 첫 번째 원소를 반환하는 함수
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}

first([1, 2, 3]);       // number | undefined
first(["a", "b", "c"]); // string | undefined

// 두 값을 쌍으로 묶는 함수
function pair<A, B>(a: A, b: B): [A, B] {
  return [a, b];
}

pair("key", 42); // [string, number]
```

---

## 제네릭 인터페이스와 타입 별칭

```typescript
interface Box<T> {
  value: T;
  label: string;
}

const numberBox: Box<number> = { value: 42, label: "숫자 상자" };
const stringBox: Box<string> = { value: "hello", label: "문자 상자" };

// 타입 별칭에서도 동일하게 사용
type Nullable<T> = T | null;
type ApiResponse<T> = {
  data: T;
  status: number;
  message: string;
};
```

---

## 제네릭 제약(Constraints)

`extends`를 사용하면 제네릭 타입이 특정 조건을 만족해야 한다고 제약할 수 있다.

```typescript
// T는 반드시 length 속성을 가져야 함
function getLength<T extends { length: number }>(arg: T): number {
  return arg.length;
}

getLength("hello");    // OK
getLength([1, 2, 3]);  // OK
getLength(42);         // 오류: number에는 length가 없음

// keyof를 이용한 제약
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const person = { name: "Alice", age: 30 };
getProperty(person, "name"); // string
getProperty(person, "age");  // number
getProperty(person, "email"); // 오류: 존재하지 않는 키
```

---

## 제네릭 클래스

```typescript
class Stack<T> {
  private items: T[] = [];

  push(item: T): void {
    this.items.push(item);
  }

  pop(): T | undefined {
    return this.items.pop();
  }

  peek(): T | undefined {
    return this.items[this.items.length - 1];
  }

  get size(): number {
    return this.items.length;
  }
}

const numStack = new Stack<number>();
numStack.push(1);
numStack.push(2);
numStack.pop(); // 2

const strStack = new Stack<string>();
strStack.push("a");
```

---

## 조건부 타입(Conditional Types)

조건부 타입은 타입 레벨에서 삼항 연산자처럼 작동한다.

```typescript
type IsString<T> = T extends string ? true : false;

type A = IsString<string>; // true
type B = IsString<number>; // false
type C = IsString<"hello">; // true (string 리터럴도 string을 extends)
```

기본 문법은 `T extends U ? X : Y` 형태다. T가 U에 할당 가능하면 X, 아니면 Y가 된다.

---

## 내장 유틸리티 타입의 조건부 타입 구현

TypeScript 내장 유틸리티 타입들이 실제로 어떻게 구현되어 있는지 살펴보면 조건부 타입을 이해하는 데 도움이 된다.

```typescript
// NonNullable 구현
type NonNullable<T> = T extends null | undefined ? never : T;

type E = NonNullable<string | null | undefined>; // string

// ReturnType 구현
type ReturnType<T extends (...args: any) => any>
  = T extends (...args: any) => infer R ? R : never;

function greet(): string { return "hello"; }
type GreetReturn = ReturnType<typeof greet>; // string

// Parameters 구현
type Parameters<T extends (...args: any) => any>
  = T extends (...args: infer P) => any ? P : never;

function add(a: number, b: number): number { return a + b; }
type AddParams = Parameters<typeof add>; // [number, number]
```

---

## infer 키워드

`infer`는 조건부 타입 안에서 타입을 추론하고 변수처럼 캡처할 때 사용한다. `extends` 절 안에서만 쓸 수 있다.

```typescript
// 배열의 원소 타입 추출
type ElementType<T> = T extends (infer E)[] ? E : never;

type NumArray = ElementType<number[]>;  // number
type StrArray = ElementType<string[]>;  // string

// Promise의 해결 타입 추출
type Awaited<T> = T extends Promise<infer U> ? Awaited<U> : T;

type Resolved = Awaited<Promise<Promise<string>>>; // string
// (실제 내장 Awaited<T>도 이와 유사하게 구현됨)

// 함수의 첫 번째 인자 타입 추출
type FirstArg<T> = T extends (first: infer F, ...rest: any[]) => any ? F : never;

type FA = FirstArg<(x: number, y: string) => void>; // number
```

---

## 분산 조건부 타입(Distributive Conditional Types)

제네릭 타입 파라미터에 유니온 타입을 넣으면 조건부 타입이 유니온의 각 멤버에 분산 적용된다.

```typescript
type ToArray<T> = T extends any ? T[] : never;

type StrOrNumArray = ToArray<string | number>;
// = ToArray<string> | ToArray<number>
// = string[] | number[]

// 분산을 막으려면 튜플로 감싸면 됨
type ToArrayNoDist<T> = [T] extends [any] ? T[] : never;

type StrOrNumArray2 = ToArrayNoDist<string | number>;
// = (string | number)[]
```

---

## 실용적인 조건부 타입 패턴

```typescript
// 특정 키만 선택적으로 만들기
type PartialBy<T, K extends keyof T> = Omit<T, K> & Partial<Pick<T, K>>;

type User = { id: number; name: string; email: string };
type PartialUser = PartialBy<User, "email">;
// = { id: number; name: string; email?: string }

// 함수 타입인지 판별
type IsFunction<T> = T extends (...args: any[]) => any ? true : false;

// 읽기 전용 키만 추출
type ReadonlyKeys<T> = {
  [K in keyof T]-?: (<U>() => U extends { [P in K]: T[P] } ? 1 : 2) extends
    (<U>() => U extends { -readonly [P in K]: T[P] } ? 1 : 2) ? never : K;
}[keyof T];

// 깊은 Partial
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K];
};

type Config = {
  server: { host: string; port: number };
  db: { url: string; name: string };
};
type PartialConfig = DeepPartial<Config>;
// server.host, server.port 등 모두 optional
```

---

## 템플릿 리터럴 타입과 조합

TypeScript 4.1부터 지원하는 템플릿 리터럴 타입과 조건부 타입을 조합하면 강력한 타입 변환이 가능하다.

```typescript
type EventName<T extends string> = `on${Capitalize<T>}`;

type Click = EventName<"click">;   // "onClick"
type Focus = EventName<"focus">;   // "onFocus"
type Change = EventName<"change">; // "onChange"

// 객체의 모든 키를 이벤트 핸들러 이름으로 변환
type EventHandlers<T extends string> = {
  [K in T as EventName<K>]: () => void;
};

type Handlers = EventHandlers<"click" | "focus" | "change">;
// { onClick: () => void; onFocus: () => void; onChange: () => void }
```

---

## 정리

| 개념 | 용도 |
|------|------|
| 제네릭 함수 | 다양한 타입에 동작하는 재사용 가능한 함수 |
| 제네릭 제약 (`extends`) | 제네릭 타입에 조건 부여 |
| `keyof`, `typeof` | 타입 레벨에서 키와 타입 참조 |
| 조건부 타입 | 타입 레벨 분기 처리 |
| `infer` | 조건부 타입 안에서 타입 추론·캡처 |
| 분산 조건부 타입 | 유니온 타입에 조건부 타입 개별 적용 |

제네릭과 조건부 타입은 TypeScript 타입 시스템의 핵심 기능이다. 유틸리티 타입들이 내부적으로 이 둘을 조합해서 구현되므로, 직접 작성해보면서 익히는 게 가장 효과적이다.
