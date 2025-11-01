---
layout: post
title: "[Daily morning study] TCP의 3-way handshake와 4-way handshake (필수)"
description: >
  #daily morning study
category: 
    - dms
    - -network
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---
# TCP의 3-way Handshake와 4-way Handshake

TCP(Transmission Control Protocol)는 신뢰성 있는 데이터 전송을 제공하는 프로토콜로, 연결 지향적입니다. 이를 위한 중요한 과정이 3-way handshake와 4-way handshake입니다. 이번 학습에서는 이 두 가지 과정을 살펴보겠습니다.

## 1. 3-way Handshake

3-way handshake는 TCP 연결을 설정하는 과정입니다. 이 과정은 클라이언트와 서버 간의 연결을 수립하기 위해 세 단계로 진행됩니다. 

### 단계별 설명

1. **SYN**: 클라이언트는 서버에 연결 요청을 위해 SYN 패킷을 전송합니다. 이때 클라이언트는 초기 순서 번호를 포함해 보냅니다.
   
   ```
   Client → Server : SYN (seq = x)
   ```

2. **SYN-ACK**: 서버는 클라이언트의 SYN을 수신한 후, 클라이언트에게 연결 수립에 대한 승인 메시지를 포함한 SYN-ACK 패킷을 응답합니다. 이때 서버도 자신의 초기 순서 번호를 포함합니다.

   ```
   Server → Client : SYN-ACK (seq = y, ack = x + 1)
   ```

3. **ACK**: 클라이언트는 서버의 SYN-ACK을 수신한 후, 연결이 설정되었다는 확인으로 ACK 패킷을 서버에게 전송합니다. 

   ```
   Client → Server : ACK (seq = x + 1, ack = y + 1)
   ```

이 과정이 끝나면 클라이언트와 서버 간의 TCP 연결이 설정됩니다.

### 시퀀스 다이어그램

아래는 3-way handshake의 흐름을 나타낸 간단한 다이어그램입니다.

```
Client              Server
   |                     |
   | -------- SYN ------>|
   |                     |
   | <----- SYN-ACK ---- |
   |                     |
   | -------- ACK ------>|
   |                     |
```

## 2. 4-way Handshake

4-way handshake는 TCP 연결을 종료하는 과정입니다. 이 과정은 네 단계로 진행됩니다.

### 단계별 설명

1. **FIN**: 클라이언트(또는 서버)는 연결 종료를 요청하기 위해 FIN 패킷을 보냅니다.

   ```
   Client → Server : FIN (seq = x)
   ```

2. **ACK**: 서버는 클라이언트의 FIN 패킷을 수신한 후, 연결 종료 요청에 대한 확인으로 ACK 패킷을 전송합니다.

   ```
   Server → Client : ACK (seq = y, ack = x + 1)
   ```

3. **FIN**: 서버는 클라이언트에게도 연결 종료를 요청하기 위해 FIN 패킷을 전송합니다.

   ```
   Server → Client : FIN (seq = y)
   ```

4. **ACK**: 클라이언트는 서버의 FIN 패킷을 수신한 후, 마지막으로 ACK 패킷을 서버에게 전송합니다.

   ```
   Client → Server : ACK (seq = x + 1, ack = y + 1)
   ```

이 과정을 모두 마치면 TCP 연결이 정상적으로 종료됩니다.

### 시퀀스 다이어그램

아래는 4-way handshake의 흐름을 나타낸 다이어그램입니다.

```
Client              Server
   |                     |
   | -------- FIN ----->|
   |                     |
   | <----- ACK --------|
   |                     |
   | <----- FIN --------|
   |                     |
   | -------- ACK ----->|
   |                     |
```

## 3. 중간 요약

- **3-way handshake**: TCP 연결을 수립하기 위해 클라이언트와 서버가 세 번의 패킷 교환을 진행합니다.
- **4-way handshake**: TCP 연결 종료를 위해 클라이언트와 서버가 네 번의 패킷 교환을 진행합니다.

## 4. 요약

TCP 프로토콜은 안정적인 연결을 위한 3-way와 4-way handshake 과정을 필수적으로 수행합니다. 3-way handshake는 데이터 통신 전에 연결을 설정하고, 4-way handshake는 통신 종료 후 안전하게 연결을 종료하는 역할을 합니다. 이 과정을 이해함으로써 TCP의 신뢰성 있는 데이터 전송 방식을 더욱 잘 이해할 수 있습니다.
