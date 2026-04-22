---
layout: post
title: "[Daily morning study] JWT vs Session 인증 방식 비교"
description: >
  #daily morning study
category: 
    - dms
    - dms-backend
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---
# JWT vs Session 인증 방식 비교

웹 애플리케이션에서 사용자 인증(Authentication)을 구현할 때 가장 많이 쓰이는 두 가지 방식이 **세션(Session) 기반 인증**과 **JWT(JSON Web Token) 기반 인증**이다. 둘 다 "로그인한 사용자를 어떻게 기억할 것인가"라는 문제를 해결하지만 접근 방식이 근본적으로 다르다.

---

## 세션 기반 인증

### 동작 방식

1. 클라이언트가 아이디/비밀번호로 로그인 요청을 보낸다.
2. 서버가 자격증명을 검증하고, 인메모리(또는 DB)에 **세션 레코드**를 생성한다.
3. 서버는 세션 ID를 생성해 **Set-Cookie** 헤더로 클라이언트에 전달한다.
4. 이후 요청마다 브라우저가 쿠키에 담긴 세션 ID를 자동으로 서버로 전송한다.
5. 서버는 세션 저장소에서 해당 ID를 조회해 사용자 정보를 확인한다.

```
[클라이언트]          [서버]              [세션 저장소]
  POST /login  ───▶  검증 후 세션 생성 ──▶ {sid: "abc123", userId: 1}
  ◀───────────────  Set-Cookie: sid=abc123
  
  GET /profile
  Cookie: sid=abc123  ───▶  세션 조회 ──▶ {userId: 1} 반환
                     ◀───  사용자 정보 응답
```

### 특징

- **상태 유지(Stateful)**: 서버가 세션 상태를 직접 관리한다.
- **즉시 무효화 가능**: 저장소에서 세션 레코드를 삭제하면 즉시 로그아웃 처리가 된다.
- **서버 메모리 사용**: 접속자가 많을수록 세션 저장소 부담이 커진다.
- **수평 확장(Scale-out) 문제**: 서버 인스턴스가 여러 개일 때 세션 정보를 공유하려면 Redis 같은 **중앙 집중식 세션 저장소**가 필요하다.

---

## JWT 기반 인증

### JWT 구조

JWT는 `.`으로 구분된 세 부분으로 이루어진다.

```
헤더(Header).페이로드(Payload).서명(Signature)
```

각 부분은 Base64URL로 인코딩된다.

```json
// Header
{ "alg": "HS256", "typ": "JWT" }

// Payload
{
  "sub": "1",          // 사용자 ID
  "name": "홍길동",
  "iat": 1713744000,   // 발급 시각
  "exp": 1713747600    // 만료 시각 (1시간 후)
}

// Signature
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  SECRET_KEY
)
```

실제 토큰은 이런 형태다:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxIiwibmFtZSI6Iu2홍길동","iat":1713744000}.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

### 동작 방식

1. 클라이언트가 로그인하면 서버가 JWT를 생성해 응답 바디로 전달한다.
2. 클라이언트는 토큰을 `localStorage` 또는 `httpOnly` 쿠키에 저장한다.
3. 이후 요청마다 `Authorization: Bearer <token>` 헤더에 토큰을 첨부한다.
4. 서버는 **서명만 검증**하면 되므로, 저장소 조회 없이 사용자 정보를 확인할 수 있다.

```
[클라이언트]          [서버]
  POST /login  ───▶  검증 후 JWT 서명 & 반환
  ◀───────────────  { token: "eyJ..." }

  GET /profile
  Authorization: Bearer eyJ...  ───▶  서명 검증만 수행
                                ◀───  사용자 정보 응답
```

### 특징

- **무상태(Stateless)**: 서버가 아무것도 저장하지 않는다. 어떤 서버 인스턴스도 같은 시크릿 키만 있으면 토큰을 검증할 수 있다.
- **수평 확장 친화적**: 별도의 세션 저장소 없이 서버를 자유롭게 늘릴 수 있다.
- **즉시 무효화가 어렵다**: 토큰은 만료 시각(exp)까지 유효하다. 강제 로그아웃을 구현하려면 블랙리스트 DB를 별도로 운영해야 해서 Stateless의 장점이 희석된다.
- **페이로드 노출**: Base64URL은 암호화가 아니므로 페이로드 내용은 누구나 디코딩할 수 있다. **민감한 정보를 페이로드에 넣으면 안 된다.**

---

## 핵심 차이 비교

| 항목 | 세션 기반 | JWT 기반 |
|------|----------|---------|
| 상태 관리 | Stateful (서버 저장) | Stateless (클라이언트 저장) |
| 저장 위치 | 서버 세션 저장소 | 클라이언트 (쿠키 / localStorage) |
| 수평 확장 | 중앙 세션 저장소 필요 | 서버 인스턴스 자유 추가 가능 |
| 강제 로그아웃 | 즉시 가능 | 토큰 블랙리스트 별도 필요 |
| 보안 위협 | 세션 하이재킹, CSRF | 토큰 탈취, XSS |
| 적합한 상황 | 모노리식 서버, 관리자 콘솔 | MSA, 모바일 앱, 외부 API |

---

## Access Token + Refresh Token 패턴

JWT의 즉시 무효화 문제를 완화하는 현실적인 방법이다.

- **Access Token**: 수명이 짧다 (5~15분). API 요청 시 사용한다.
- **Refresh Token**: 수명이 길다 (7~30일). DB에 저장하고, Access Token 재발급 시 사용한다.

```
클라이언트                         서버
   │  POST /login                  │
   │ ─────────────────────────────▶│  검증 후 두 토큰 발급
   │◀─────────────────────────────│  AccessToken(15분) + RefreshToken(7일)
   │                               │
   │  API 요청 + AccessToken       │
   │ ─────────────────────────────▶│  서명 검증
   │                               │
   │  AccessToken 만료 후          │
   │  POST /auth/refresh           │
   │  + RefreshToken               │
   │ ─────────────────────────────▶│  DB에서 RefreshToken 확인 후
   │◀─────────────────────────────│  새 AccessToken 발급
```

Refresh Token을 DB에 저장하기 때문에 토큰을 DB에서 삭제하는 것만으로 로그아웃 처리가 가능하다. 완전한 Stateless는 아니지만, 검증 비용이 큰 Access Token 검증을 DB 없이 처리할 수 있어 실용적인 절충안이다.

---

## 보안 고려 사항

### 세션 방식
- `httpOnly`, `Secure`, `SameSite=Strict` 쿠키 설정으로 CSRF, XSS를 방어한다.
- CSRF 토큰을 함께 사용하는 것이 권장된다.

### JWT 방식
- `localStorage`에 저장하면 XSS 취약점에 노출된다. `httpOnly` 쿠키에 저장하는 것이 더 안전하다.
- 페이로드에 민감 정보(비밀번호, 신용카드 번호 등)를 절대 담지 않는다.
- 알고리즘을 `none`으로 설정하는 공격에 대비해 서버에서 알고리즘을 명시적으로 지정해야 한다.

```js
// 검증 시 알고리즘 명시 (Node.js jsonwebtoken 라이브러리 예시)
jwt.verify(token, SECRET_KEY, { algorithms: ['HS256'] });
```

---

## 선택 기준 정리

- **세션 방식**이 유리한 경우: 즉각적인 접근 취소가 필요한 경우, 단일 서버 혹은 소규모 서비스, 관리자 도구처럼 보안이 매우 중요한 경우.
- **JWT 방식**이 유리한 경우: 마이크로서비스, 모바일 앱, 외부 파트너에게 API를 제공하는 경우, 서버를 자주 스케일아웃하는 경우.

실제 프로덕션에서는 순수 JWT 하나만 쓰기보다 **Access Token + Refresh Token** 조합으로 보안과 확장성을 함께 챙기는 구조가 가장 흔하다.
