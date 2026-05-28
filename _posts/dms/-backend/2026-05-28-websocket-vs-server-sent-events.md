---
layout: post
title: "[Daily morning study] WebSocket vs Server-Sent Events"
description: >
  #daily morning study
category: 
    - dms
    - -backend
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## WebSocket과 SSE, 언제 무엇을 쓸까

실시간 통신이 필요한 상황이면 WebSocket 아니면 SSE(Server-Sent Events) 중 하나를 고르게 된다. 둘 다 HTTP 기반이지만 방향성과 구조가 다르다.

---

## WebSocket

WebSocket은 **양방향(full-duplex)** 통신 프로토콜이다. 클라이언트가 HTTP 업그레이드 요청을 보내면, 서버가 `101 Switching Protocols`로 응답하면서 TCP 연결이 WebSocket 연결로 전환된다. 연결이 수립된 이후에는 클라이언트와 서버 모두 언제든 메시지를 보낼 수 있다.

```
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

→ HTTP/1.1 101 Switching Protocols
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

이 핸드셰이크가 끝나면 HTTP 연결이 종료되고, 동일한 TCP 소켓 위에서 WebSocket 프레임을 주고받는다.

### Node.js 서버 예시 (ws 라이브러리)

```javascript
const WebSocket = require('ws');
const wss = new WebSocket.Server({ port: 8080 });

wss.on('connection', (ws) => {
  ws.on('message', (message) => {
    console.log('받은 메시지:', message.toString());
    ws.send('에코: ' + message.toString());
  });

  ws.on('close', () => {
    console.log('클라이언트 연결 종료');
  });
});
```

### 클라이언트 예시

```javascript
const ws = new WebSocket('ws://localhost:8080');

ws.onopen = () => {
  ws.send('안녕하세요');
};

ws.onmessage = (event) => {
  console.log('서버로부터:', event.data);
};

ws.onclose = () => {
  console.log('연결 끊김 — 재연결 로직 필요');
};
```

WebSocket은 자동 재연결을 제공하지 않는다. 연결이 끊기면 직접 재연결 로직을 구현해야 한다.

---

## Server-Sent Events (SSE)

SSE는 **서버 → 클라이언트 단방향** 스트리밍이다. 별도 프로토콜 업그레이드 없이 `Content-Type: text/event-stream`을 사용하는 일반 HTTP 롱-리빙 연결이다. 클라이언트는 `EventSource` API로 구독하고, 브라우저가 자동으로 재연결을 처리한다.

### Express 서버 예시

```javascript
app.get('/events', (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');

  // 이벤트 ID를 보내면 재연결 시 클라이언트가 Last-Event-ID 헤더로 보내줌
  let id = 0;

  const timer = setInterval(() => {
    res.write(`id: ${id++}\n`);
    res.write(`event: update\n`);
    res.write(`data: ${JSON.stringify({ time: Date.now() })}\n\n`);
  }, 1000);

  req.on('close', () => {
    clearInterval(timer);
  });
});
```

SSE 이벤트 형식은 `\n\n`으로 각 이벤트를 구분한다. 필드는 `data:`, `event:`, `id:`, `retry:` 네 가지다.

### 클라이언트 예시

```javascript
const eventSource = new EventSource('/events');

eventSource.addEventListener('update', (event) => {
  const data = JSON.parse(event.data);
  console.log('시간:', data.time);
});

eventSource.onerror = () => {
  // 브라우저가 자동 재연결을 시도함
  console.log('연결 오류 — 자동 재시도 중');
};

// 더 이상 필요 없으면 명시적으로 닫기
// eventSource.close();
```

---

## 핵심 비교

| 항목 | WebSocket | SSE |
|------|-----------|-----|
| 통신 방향 | 양방향 | 서버 → 클라이언트 단방향 |
| 프로토콜 | WS / WSS | HTTP / HTTPS |
| 데이터 형식 | 텍스트 + 바이너리 | 텍스트만 (UTF-8) |
| 자동 재연결 | 없음 (직접 구현) | 있음 (브라우저 내장) |
| 방화벽 통과 | 프록시 설정 따라 다름 | 표준 HTTP라 대체로 무난 |
| 브라우저 지원 | 매우 넓음 | 넓음 (IE 제외) |
| HTTP/2 멀티플렉싱 | 불가 | 가능 |
| 동시 연결 제한 | 없음 | HTTP/1.1에서 도메인당 6개 |

HTTP/2를 쓰면 SSE의 동시 연결 6개 제한이 사라진다. 하나의 TCP 연결에서 여러 스트림을 멀티플렉싱하기 때문이다.

---

## 수평 확장 시 고려사항

### WebSocket

WebSocket 연결은 특정 서버 인스턴스와 맺어진다. 여러 서버로 스케일아웃할 때 같은 클라이언트가 매번 동일한 서버로 라우팅되도록 **스티키 세션**을 설정하거나, Redis Pub/Sub 같은 공유 메시지 브로커를 두어 서버 간 메시지를 동기화해야 한다.

```
클라이언트 A ──→ 서버 1 ──→ Redis Pub/Sub ──→ 서버 2 ──→ 클라이언트 B
```

### SSE

SSE도 롱-리빙 HTTP 연결이라 동일한 문제가 있다. 마찬가지로 Redis나 메시지 큐를 통해 이벤트를 브로드캐스트하는 구조가 일반적이다.

---

## 어떤 걸 선택할까

**WebSocket이 적합한 경우**

- 채팅, 협업 편집기처럼 클라이언트도 서버로 데이터를 자주 보내야 할 때
- 온라인 게임, 주식 거래 플랫폼처럼 매우 낮은 레이턴시가 필요할 때
- 바이너리 데이터(이미지, 파일 조각)를 실시간으로 전송해야 할 때

**SSE가 적합한 경우**

- 뉴스 피드, 알림, 라이브 스코어처럼 서버에서 클라이언트로만 데이터가 흐를 때
- 구현을 최대한 단순하게 유지하고 싶을 때 (추가 라이브러리 없이 표준 HTTP로 동작)
- 재연결 처리를 브라우저에게 맡기고 싶을 때
- 로그 스트리밍, 빌드 진행 상태 등 단순 텍스트 스트리밍에 적합

WebSocket은 양방향이 필요할 때 쓰고, 서버 푸시만 필요하다면 SSE가 구현이 더 단순하고 표준 HTTP 인프라와 친화적이다.
