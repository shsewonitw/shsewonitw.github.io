---
layout: post
title: "[Daily morning study] TypeScript의 Utility Types (Partial, Pick, Omit 등)"
description: >
  #daily morning study
category: 
    - dms
    - -front
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---
# TypeScript의 Utility Types

TypeScript는 개발자가 더욱 효율적으로 코드 작성할 수 있게 돕는 여러 다양한 유틸리티 타입을 제공합니다. 이 가이드에서는 `Partial`, `Pick`, `Omit`, `Record`, `Exclude`, `Extract`와 같은 주요 유틸리티 타입에 대해 설명하겠습니다.

## 1. Partial<T>

`Partial<T>` 타입은 주어진 타입 `T`의 모든 프로퍼티를 선택적(optional)으로 만들어 줍니다. 이 유틸리티는 특정 객체의 일부 프로퍼티만을 업데이트할 때 유용합니다.

### 예시

```typescript
interface User {
  id: number;
  name: string;
  email: string;
}

const updateUser = (id: number, userUpdates: Partial<User>) => {
  // 업데이트 로직
};

updateUser(1, { name: "새 이름" });
```

위 코드에서 `userUpdates`는 `User` 인터페이스의 선택적인 프로퍼티들만 포함할 수 있습니다.

## 2. Pick<T, K>

`Pick<T, K>` 타입은 주어진 타입 `T`에서 특정 프로퍼티 `K`만 선택하여 새로운 타입을 생성합니다. 주로 큰 객체에서 일부 프로퍼티만 필요한 경우에 유용합니다.

### 예시

```typescript
interface User {
  id: number;
  name: string;
  email: string;
}

type UserPreview = Pick<User, "id" | "name">;

const user1: UserPreview = {
  id: 1,
  name: "사용자1",
};
```

위 예시에서는 `UserPreview` 타입이 `User` 타입에서 `id`와 `name` 프로퍼티만 선택한 형태입니다.

## 3. Omit<T, K>

`Omit<T, K>` 타입은 `Pick`과는 반대로, 특정 프로퍼티 `K`를 제외한 새로운 타입을 만듭니다. 

### 예시

```typescript
interface User {
  id: number;
  name: string;
  email: string;
}

type UserWithoutEmail = Omit<User, "email">;

const user2: UserWithoutEmail = {
  id: 2,
  name: "사용자2",
};
```

위 예시에서는 `UserWithoutEmail` 타입이 `User` 타입에서 `email` 프로퍼티를 제외한 형태입니다.

## 4. Record<K, T>

`Record<K, T>` 타입은 주어진 키 `K`와 값 `T`를 가진 객체 타입을 생성합니다. 주로 여러 개의 값이 동일한 타입일 때 유용합니다.

### 예시

```typescript
type UserRoles = "admin" | "editor" | "viewer";
type RolePermissions = Record<UserRoles, string[]>;

const permissions: RolePermissions = {
  admin: ["read", "write", "delete"],
  editor: ["read", "write"],
  viewer: ["read"],
};
```

위 코드에서는 `UserRoles` 타입의 각 역할에 대해 권한을 정의하는 객체를 생성하였습니다.

## 5. Exclude<T, U>

`Exclude<T, U>`는 주어진 타입 `T`에서 `U`에 해당하는 타입을 제외한 새로운 타입을 생성합니다.

### 예시

```typescript
type AllNumbers = number | string | boolean;
type OnlyNumbers = Exclude<AllNumbers, string | boolean>; // number만 남습니다.
```

위 코드에서 `OnlyNumbers`는 `number` 타입만 포함합니다.

## 6. Extract<T, U>

`Extract<T, U>`는 주어진 타입 `T` 중에서 `U`의 서브타입을 추출하여 새로운 타입을 생성합니다.

### 예시

```typescript
type AllNumbers = number | string | boolean;
type StringOrNumber = Extract<AllNumbers, string | number>; // string | number
```

위 코드에서 `StringOrNumber`는 `string`과 `number` 타입만 포함합니다.

## 마무리

TypeScript의 유틸리티 타입들은 타입 관리를 훨씬 수월하게 만들어 주며, 코드의 가독성을 높이는 데 기여합니다. 이런 유틸리티 타입들을 잘 활용하면 더욱 효율적이고 안전한 코드를 작성할 수 있습니다. 각 유틸리티 타입의 사용 사례와 함께 코드를 작성해보며 익혀보세요!
