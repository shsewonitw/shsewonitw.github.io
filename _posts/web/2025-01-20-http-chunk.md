---
layout: post
title: "[Web] HTTP Streaming"
subtitle: "GPT는 어떻게 데이터를 쪼개서 보내는가"
category: 
    - devlog
    - web
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/cf27833e-0910-4772-a185-d10ff31d2622)

---

# ChatGPT의 실시간 응답 원리 — HTTP Chunked Transfer Encoding

GPT API를 사용하면 응답이 한번에 오지 않고, 마치 "생각하는 것처럼" 조금씩 도착합니다.  
이것은 바로 **HTTP Chunked Transfer Encoding** 덕분입니다.

## HTTP Chunked Transfer Encoding 이란?

- HTTP 응답을 한 번에 전송하지 않고, **작은 조각(chunk)** 으로 나누어 전송하는 방식
- GPT API는 이 방식을 사용해 사용자가 더 빠르게 응답을 확인할 수 있도록 합니다.

## 실제 예시

```http
HTTP/1.1 200 OK
Transfer-Encoding: chunked
Content-Type: application/json
```

```json
{
  "delta": { "content": "Hello" }
}
```

## WebSocket과의 차이점

| 항목 | HTTP Streaming | WebSocket |
| ---- | -------------- | --------- |
| 데이터 방향 | 서버 → 클라이언트 (단방향) | 클라이언트 ↔ 서버 (양방향) |
| 사용 예 | GPT 응답, 로그 | 채팅, 협업 툴 |

## 결론

GPT는 WebSocket 대신 HTTP Streaming을 통해 효율적이면서도 친숙한 방식으로 실시간 응답을 제공합니다.
