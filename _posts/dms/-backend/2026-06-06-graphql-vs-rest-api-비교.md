---
layout: post
title: "[Daily morning study] GraphQL vs REST API 비교"
description: >
  #daily morning study
category: 
    - dms
    - dms-backend
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## REST API 기본 개념

REST(Representational State Transfer)는 HTTP 프로토콜 기반의 아키텍처 스타일이다. 각 리소스는 고유한 URL로 표현되고, HTTP 메서드로 CRUD 작업을 수행한다.

```
GET    /users       → 사용자 목록
GET    /users/1     → 특정 사용자
POST   /users       → 사용자 생성
PUT    /users/1     → 사용자 수정
DELETE /users/1     → 사용자 삭제
```

## GraphQL 기본 개념

GraphQL은 Facebook(현 Meta)이 2015년에 공개한 쿼리 언어이자 런타임이다. 클라이언트가 필요한 데이터의 구조를 직접 정의해서 요청한다.

```graphql
query {
  user(id: "1") {
    name
    email
    posts {
      title
      createdAt
    }
  }
}
```

단일 엔드포인트(`/graphql`)로 모든 요청을 처리한다는 점이 REST와 가장 큰 구조적 차이다.

---

## 핵심 차이점 비교

| 항목 | REST | GraphQL |
|------|------|---------|
| 엔드포인트 | 리소스마다 다름 | 단일 엔드포인트 |
| 응답 형태 결정권 | 서버 | 클라이언트 |
| Over-fetching | 자주 발생 | 해결됨 |
| Under-fetching | 자주 발생 | 해결됨 |
| HTTP 캐싱 | 기본 지원 | 별도 구현 필요 |
| 파일 업로드 | 간단 | 추가 라이브러리 필요 |
| 학습 곡선 | 낮음 | 상대적으로 높음 |
| 버전 관리 | /v1/, /v2/ 방식 | 스키마 진화로 처리 |

---

## Over-fetching과 Under-fetching

REST API에서 자주 발생하는 두 가지 비효율이다.

**Over-fetching**: 필요 이상의 데이터를 받는 것

사용자 이름만 필요한데 `/users/1`을 호출하면 name, email, age, address, createdAt 등 모든 필드가 응답으로 온다.

**Under-fetching**: 한 번의 요청으로 필요한 데이터를 다 못 받는 것

사용자 정보와 게시글 목록을 함께 표시하려면 두 번 호출해야 한다.

```
GET /users/1       → 사용자 정보
GET /users/1/posts → 게시글 목록
```

이 패턴이 반복되면 N+1 문제로 이어진다. GraphQL은 한 번의 쿼리로 필요한 필드만 정확히 가져와 두 문제를 동시에 해결한다.

---

## GraphQL 핵심 구성 요소

### Schema와 Type System

GraphQL은 강타입 시스템을 갖는다. 서버는 스키마로 데이터 형태를 미리 정의하고, 클라이언트는 그 범위 안에서 자유롭게 쿼리를 구성한다.

```graphql
type User {
  id: ID!
  name: String!
  email: String!
  posts: [Post!]!
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
}

type Query {
  user(id: ID!): User
  users: [User!]!
}

type Mutation {
  createUser(name: String!, email: String!): User!
  deleteUser(id: ID!): Boolean!
}
```

`!`는 null 불가를 의미한다.

### Query, Mutation, Subscription

- **Query**: 데이터 조회 (REST의 GET에 해당)
- **Mutation**: 데이터 변경 (POST, PUT, DELETE에 해당)
- **Subscription**: 실시간 데이터 구독 (WebSocket 기반)

```graphql
subscription {
  messageAdded(roomId: "1") {
    content
    author {
      name
    }
  }
}
```

Subscription은 REST가 기본적으로 지원하지 않는 실시간 기능을 스키마 수준에서 정의할 수 있게 해준다.

### Resolver

클라이언트가 요청한 각 필드에 대해 실제 데이터를 가져오는 함수다.

```javascript
const resolvers = {
  Query: {
    user: (_, { id }) => User.findById(id),
    users: () => User.findAll(),
  },
  User: {
    posts: (user) => Post.findByAuthorId(user.id),
  },
};
```

각 필드마다 resolver가 실행되므로, 중첩된 데이터 구조에서는 N+1 문제가 생길 수 있다.

---

## N+1 문제와 DataLoader

사용자 10명의 게시글을 한 번에 가져오는 쿼리를 실행하면:

1. `users` 쿼리 실행 → 1번 DB 호출
2. 각 User의 `posts` resolver 실행 → 10번 DB 호출
3. 총 11번 DB 쿼리 발생

이를 해결하기 위해 **DataLoader**를 사용한다. 요청을 배치(batch)로 묶어 단 1번의 DB 쿼리로 처리한다.

```javascript
const postLoader = new DataLoader(async (userIds) => {
  const posts = await Post.findByAuthorIds(userIds); // IN 쿼리 1번
  return userIds.map(id => posts.filter(p => p.authorId === id));
});

// User resolver에서
User: {
  posts: (user) => postLoader.load(user.id),
}
```

DataLoader는 같은 이벤트 루프 내에서 발생하는 `load()` 호출을 자동으로 묶어서 처리한다.

---

## REST가 더 나은 경우

- **단순한 CRUD API**: 복잡한 데이터 관계가 없는 경우
- **파일 업로드**: multipart/form-data 처리가 간단
- **HTTP 캐싱 적극 활용**: CDN, 브라우저 캐시를 그대로 사용
- **공개 API**: 다양한 외부 클라이언트가 사용하는 퍼블릭 API
- **팀이 REST에 익숙하고 학습 비용을 줄여야 하는 경우**

## GraphQL이 더 나은 경우

- **모바일 앱**: 네트워크 비용 절감, 필요한 필드만 수신
- **복잡한 데이터 관계**: 여러 리소스를 한 쿼리로 가져와야 할 때
- **다양한 클라이언트**: 웹, 모바일, IoT가 각각 다른 형태의 데이터를 필요로 할 때
- **빠른 프론트엔드 개발**: 백엔드 변경 없이 클라이언트가 원하는 데이터 구조를 직접 정의
- **실시간 기능**: Subscription으로 WebSocket 기반 실시간 통신

---

## 두 방식을 함께 쓰는 하이브리드 패턴

실무에서는 REST와 GraphQL을 동시에 사용하는 경우도 많다.

- **인증, 파일 업로드** → REST (`/auth/login`, `/upload`)
- **데이터 조회 및 변경** → GraphQL (`/graphql`)

캐싱이 중요한 퍼블릭 콘텐츠는 REST로, 복잡한 데이터를 다루는 내부 API는 GraphQL로 처리하는 방식이다.

---

## 정리

REST와 GraphQL은 각자 잘 맞는 상황이 다르다. REST는 단순하고 HTTP 표준과 잘 맞는 반면, GraphQL은 클라이언트 요구사항이 다양하고 데이터 관계가 복잡할 때 강점을 발휘한다. 선택 기준은 프로젝트 복잡도, 팀 역량, 클라이언트 다양성이다.
