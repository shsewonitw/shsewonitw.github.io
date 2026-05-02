---
layout: post
title: "[Daily morning study] 웹소켓 프로토콜 동작 방식"
description: >
  #daily morning study
category: 
    - dms
    - dms-network
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## WebSocket이란

HTTP는 기본적으로 요청-응답(Request-Response) 모델이다. 클라이언트가 요청해야만 서버가 응답할 수 있고, 연결은 응답 후 끊긴다. 실시간 데이터가 필요한 애플리케이션에서는 이 구조가 비효율적이다.

WebSocket은 이 문제를 해결하기 위해 만들어진 프로토콜이다. 최초 연결을 HTTP로 시작하지만, 이후 연결을 TCP 위에서 유지하면서 클라이언트와 서버가 양방향으로 자유롭게 메시지를 주고받을 수 있다. RFC 6455에 정의되어 있다.

---

## HTTP Polling과의 차이

WebSocket이 등장하기 전에는 실시간처럼 보이게 하기 위해 Polling 방식을 썼다.

| 방식 | 동작 | 문제점 |
|------|------|--------|
| Short Polling | 클라이언트가 일정 주기로 요청 반복 | 불필요한 요청 과다, 지연 발생 |
| Long Polling | 서버가 데이터 생길 때까지 응답 보류 | 연결 점유, 서버 부하 |
| Server-Sent Events (SSE) | 서버→클라이언트 단방향 스트림 | 클라이언트→서버 전송 불가 |
| WebSocket | 양방향 지속 연결 | 초기 핸드셰이크 오버헤드 |

WebSocket은 연결을 한 번 맺으면 계속 유지하기 때문에 반복적인 HTTP 오버헤드가 없다.

---

## 연결 수립 과정 (핸드셰이크)

WebSocket 연결은 HTTP Upgrade 요청으로 시작된다.

### 1. 클라이언트 → 서버: Upgrade 요청

```http
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

- `Upgrade: websocket` — 프로토콜을 WebSocket으로 전환 요청
- `Sec-WebSocket-Key` — 클라이언트가 생성한 랜덤 Base64 인코딩 값. 서버가 올바른 WebSocket 서버임을 증명하는 데 쓰인다.

### 2. 서버 → 클라이언트: 101 Switching Protocols

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

- 상태 코드 `101`은 프로토콜 전환 승인을 의미
- `Sec-WebSocket-Accept` — 클라이언트의 `Sec-WebSocket-Key`에 고정 GUID(`258EAFA5-E914-47DA-95CA-C5AB0DC85B11`)를 붙인 뒤 SHA-1 해싱하고 Base64로 인코딩한 값

이 과정이 완료되면 HTTP 연결이 WebSocket 연결로 업그레이드되고, 이후부터는 WebSocket 프레임 형식으로 통신한다.

---

## 데이터 전송: 프레임 구조

WebSocket은 메시지를 **프레임(Frame)** 단위로 전송한다. 하나의 메시지가 여러 프레임으로 나뉠 수 있다.

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |                               |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+-------------------------------+
```

주요 필드:

- **FIN** — 마지막 프레임이면 1
- **opcode** — 프레임 타입 (0x1: 텍스트, 0x2: 바이너리, 0x8: 연결 종료, 0x9: ping, 0xA: pong)
- **MASK** — 클라이언트→서버 방향은 반드시 마스킹해야 함 (RFC 규정)
- **Payload len** — 페이로드 길이

클라이언트에서 서버로 보내는 프레임은 반드시 마스킹해야 한다. 서버에서 클라이언트로 보내는 프레임은 마스킹하지 않는다.

---

## 연결 종료

양쪽 모두 Close 프레임(opcode `0x8`)을 보내서 연결을 종료할 수 있다.

```
클라이언트 → 서버: Close Frame (코드 1000: 정상 종료)
서버 → 클라이언트: Close Frame (에코)
TCP 연결 종료
```

상태 코드 종류:

| 코드 | 의미 |
|------|------|
| 1000 | 정상 종료 |
| 1001 | 엔드포인트 떠남 (페이지 이동 등) |
| 1006 | 비정상 종료 (Close 프레임 없이 끊김) |
| 1011 | 서버 내부 오류 |

---

## Ping / Pong

연결이 살아있는지 확인하는 heartbeat 메커니즘이다.

- 서버가 `ping` 프레임 전송
- 클라이언트가 `pong` 프레임으로 응답
- 클라이언트도 ping을 보낼 수 있음

응답이 없으면 연결이 끊긴 것으로 판단하고 정리한다.

---

## JavaScript에서 사용하기

브라우저는 `WebSocket` API를 기본 제공한다.

```javascript
const ws = new WebSocket('wss://example.com/chat');

ws.onopen = () => {
  console.log('연결 성공');
  ws.send(JSON.stringify({ type: 'join', room: 'general' }));
};

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log('수신:', data);
};

ws.onclose = (event) => {
  console.log('연결 종료:', event.code, event.reason);
};

ws.onerror = (error) => {
  console.error('에러:', error);
};

// 연결 종료
ws.close(1000, 'Done');
```

URL 스킴:
- `ws://` — 비암호화 (HTTP와 동일한 포트 80)
- `wss://` — TLS 암호화 (HTTPS와 동일한 포트 443)

실무에서는 반드시 `wss://`를 써야 한다.

---

## Node.js 서버 예시 (ws 라이브러리)

```javascript
const { WebSocketServer } = require('ws');

const wss = new WebSocketServer({ port: 8080 });

wss.on('connection', (ws, req) => {
  console.log('클라이언트 연결:', req.socket.remoteAddress);

  ws.on('message', (data) => {
    // 모든 클라이언트에게 브로드캐스트
    wss.clients.forEach((client) => {
      if (client.readyState === WebSocket.OPEN) {
        client.send(data.toString());
      }
    });
  });

  ws.on('close', () => {
    console.log('클라이언트 연결 종료');
  });
});
```

---

## WebSocket vs SSE (Server-Sent Events)

| 항목 | WebSocket | SSE |
|------|-----------|-----|
| 방향 | 양방향 | 서버→클라이언트 단방향 |
| 프로토콜 | 별도 (RFC 6455) | HTTP 위에서 동작 |
| 브라우저 지원 | 모든 현대 브라우저 | 모든 현대 브라우저 |
| 자동 재연결 | 직접 구현해야 함 | 브라우저가 자동 처리 |
| 메시지 형식 | 바이너리/텍스트 모두 가능 | 텍스트만 |
| 적합한 용도 | 채팅, 게임, 협업 툴 | 알림, 피드, 대시보드 |

클라이언트→서버 전송이 필요 없는 경우(알림, 실시간 피드)라면 SSE가 더 단순하고 HTTP 인프라를 그대로 활용할 수 있다.

---

## 실무 주의사항

**로드 밸런서 설정**
WebSocket은 연결을 유지하기 때문에 로드 밸런서에서 sticky session(세션 고정)을 설정하거나, 연결 상태를 Redis 같은 외부 저장소로 관리해야 한다.

**스케일 아웃 문제**
서버가 여러 대일 때 서버 A에 연결된 클라이언트와 서버 B에 연결된 클라이언트가 서로 메시지를 주고받으려면 Redis Pub/Sub 같은 브로커가 필요하다.

**reconnect 처리**
네트워크 불안정으로 연결이 끊길 수 있다. 클라이언트 측에서 exponential backoff로 재연결 로직을 구현해야 한다.

```javascript
function connect() {
  const ws = new WebSocket('wss://example.com/ws');
  let retryDelay = 1000;

  ws.onclose = () => {
    setTimeout(() => {
      retryDelay = Math.min(retryDelay * 2, 30000);
      connect();
    }, retryDelay);
  };

  ws.onopen = () => {
    retryDelay = 1000; // 성공 시 초기화
  };
}
```

**메모리 누수**
연결된 클라이언트 목록을 관리할 때 종료된 연결을 제대로 정리하지 않으면 메모리 누수가 발생한다. `readyState`를 확인하거나 `close` 이벤트에서 목록에서 제거해야 한다.
