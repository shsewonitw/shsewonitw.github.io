---
layout: post
title: "[Daily morning study] ICMP 프로토콜 동작 방식과 ping, traceroute 활용"
description: >
  #daily morning study
category: 
    - dms
    - -network
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## ICMP (Internet Control Message Protocol)

ICMP는 IP 네트워크에서 오류 보고와 진단 목적으로 사용하는 프로토콜이다. TCP나 UDP처럼 데이터를 전송하는 것이 목적이 아니라, 네트워크 상태를 알리는 제어 메시지를 전달하는 데 쓰인다.

IP 헤더에 `Protocol: 1`로 캡슐화되며, IP 계층(3계층)에서 동작한다. 하지만 엄밀히는 IP의 일부로 취급되어 OSI 모델의 네트워크 계층에 속한다.

---

## ICMP 메시지 구조

ICMP 메시지는 8바이트 헤더 + 가변 데이터 부분으로 구성된다.

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|      Type     |      Code     |           Checksum            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Rest of Header                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          Data                                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- **Type**: 메시지 종류 (0, 3, 8, 11 등)
- **Code**: 해당 Type의 세부 코드
- **Checksum**: 오류 검출용

---

## 주요 ICMP 메시지 타입

| Type | Code | 이름 | 설명 |
|------|------|------|------|
| 0 | 0 | Echo Reply | ping 응답 |
| 3 | 0~15 | Destination Unreachable | 목적지 도달 불가 |
| 8 | 0 | Echo Request | ping 요청 |
| 11 | 0 | Time Exceeded (TTL) | TTL 만료 |
| 11 | 1 | Time Exceeded (Fragment) | 단편 재조립 시간 초과 |
| 12 | 0 | Parameter Problem | IP 헤더 오류 |

### Destination Unreachable의 Code 값

| Code | 의미 |
|------|------|
| 0 | Network Unreachable (라우팅 실패) |
| 1 | Host Unreachable (호스트 응답 없음) |
| 2 | Protocol Unreachable (해당 프로토콜 미지원) |
| 3 | Port Unreachable (포트 닫혀 있음) |
| 4 | Fragmentation Needed (단편화 필요한데 DF 비트 설정됨) |

---

## ping 동작 방식

`ping`은 ICMP Echo Request(Type 8)를 보내고 Echo Reply(Type 0)를 받아서 통신 가능 여부와 RTT(Round-Trip Time)를 측정한다.

```
[클라이언트]  →  ICMP Echo Request (Type=8, Code=0)  →  [서버]
[클라이언트]  ←  ICMP Echo Reply   (Type=0, Code=0)  ←  [서버]
```

### ping 패킷 구조

```
ICMP Echo Request:
  Type: 8
  Code: 0
  Identifier: 12345   (프로세스 식별용)
  Sequence Number: 1  (요청 순서)
  Data: 임의 바이트   (보통 타임스탬프 포함)
```

클라이언트는 `Identifier`와 `Sequence Number`로 어떤 요청에 대한 응답인지 매칭한다.

### ping 명령어 예시

```bash
ping google.com

# 출력 예시:
PING google.com (142.250.196.110): 56 data bytes
64 bytes from 142.250.196.110: icmp_seq=0 ttl=117 time=4.823 ms
64 bytes from 142.250.196.110: icmp_seq=1 ttl=117 time=5.021 ms

--- google.com ping statistics ---
2 packets transmitted, 2 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 4.823/4.922/5.021/0.099 ms
```

- **ttl**: 패킷이 거친 라우터 수를 역산할 수 있음 (초기 TTL - 수신 TTL = 홉 수)
- **time**: RTT (ms 단위)
- **icmp_seq**: 순서 번호로 패킷 손실 여부 확인 가능

---

## TTL (Time to Live)과 ICMP

IP 헤더의 TTL 필드는 패킷이 무한 루프를 돌지 않도록 라우터를 거칠 때마다 1씩 감소한다. TTL이 0이 되면 해당 라우터가 패킷을 버리고 **ICMP Time Exceeded (Type=11, Code=0)** 메시지를 출발지에게 보낸다.

```
[클라이언트] → 패킷(TTL=64) → [라우터A] → 패킷(TTL=63) → [라우터B]
                                                               ↓ TTL=0
                                         ICMP Time Exceeded ←←←
```

---

## traceroute 동작 방식

`traceroute`(Linux/Mac) 또는 `tracert`(Windows)는 목적지까지 경로상의 모든 라우터를 찾아내는 도구다. ICMP의 Time Exceeded 메시지를 활용한다.

### 동작 원리

TTL을 1부터 시작해서 매번 1씩 늘려가며 패킷을 보낸다.

```
1단계: TTL=1 → 첫 번째 라우터에서 TTL 만료 → ICMP Time Exceeded 수신 → 첫 번째 라우터 IP 확인
2단계: TTL=2 → 두 번째 라우터에서 TTL 만료 → ICMP Time Exceeded 수신 → 두 번째 라우터 IP 확인
3단계: TTL=3 → ...
...
N단계: TTL=N → 목적지 도달 → ICMP Echo Reply (또는 UDP Port Unreachable) 수신
```

### Linux/Mac traceroute

Linux와 Mac에서 기본 `traceroute`는 **UDP 패킷**을 사용한다. 목적지에서 닫혀 있는 높은 포트(33434+)로 UDP를 보내면, 목적지가 Port Unreachable (ICMP Type=3, Code=3)을 돌려보낸다.

```bash
traceroute google.com

# 출력 예시:
traceroute to google.com (142.250.196.110), 64 hops max, 52 byte packets
 1  192.168.1.1 (192.168.1.1)  1.234 ms  1.102 ms  0.987 ms
 2  10.0.0.1 (10.0.0.1)  5.431 ms  5.123 ms  4.987 ms
 3  * * *
 4  142.250.196.110 (142.250.196.110)  8.901 ms  8.765 ms  8.432 ms
```

- `* * *`: 해당 라우터가 ICMP를 차단하거나 응답하지 않음
- 각 줄에 RTT 3개가 표시되는 이유는 각 TTL 값으로 3번씩 패킷을 보내기 때문

### Windows tracert

Windows는 ICMP Echo Request를 사용한다 (TTL을 올려가며 보냄).

```
tracert google.com
```

### traceroute 옵션

```bash
# ICMP 방식으로 강제 사용 (-I)
traceroute -I google.com

# TCP SYN 방식 (포트 80)
traceroute -T -p 80 google.com

# 최대 홉 수 제한
traceroute -m 15 google.com

# 각 홉에서 시도 횟수 변경
traceroute -q 1 google.com
```

---

## ICMPv6

IPv6에서는 ICMPv6가 기존 ICMP 역할 + ARP 역할까지 담당한다. IPv4에서 ARP로 처리하던 MAC 주소 해석을 ICMPv6의 **NDP(Neighbor Discovery Protocol)**가 처리한다.

주요 ICMPv6 메시지:
- **Type 133**: Router Solicitation (라우터 탐색 요청)
- **Type 134**: Router Advertisement (라우터 정보 광고)
- **Type 135**: Neighbor Solicitation (ARP Request에 해당)
- **Type 136**: Neighbor Advertisement (ARP Reply에 해당)
- **Type 128**: Echo Request
- **Type 129**: Echo Reply

---

## ICMP와 방화벽

ICMP는 보안상 이유로 방화벽에서 차단하는 경우가 많다. 하지만 무작정 전부 막으면 문제가 생긴다.

### 차단해도 되는 것

- Echo Request/Reply (ping): 보안 스캔에 활용될 수 있으므로 외부에서 오는 ping을 막는 경우 多

### 절대 막으면 안 되는 것

- **Destination Unreachable (Type=3, Code=4)**: Path MTU Discovery에 필수. 이걸 막으면 MTU 크기를 초과하는 패킷이 전달되지 않아 TCP 연결이 멈추는 "블랙홀" 현상이 발생
- **Time Exceeded (Type=11)**: traceroute 동작에 필요

### PMTUD (Path MTU Discovery)

경로상 최소 MTU를 자동으로 알아내는 메커니즘. IP 헤더의 DF(Don't Fragment) 비트를 1로 설정하고 패킷을 보낼 때, 중간 라우터가 MTU를 초과한다고 판단하면 ICMP Type=3, Code=4 (Fragmentation Needed)를 보내며 적절한 MTU 값을 알려준다.

```
[클라이언트] → 패킷(MTU=1500, DF=1) → [라우터: MTU=1400]
                                         ↓
[클라이언트] ← ICMP Type=3, Code=4 (Next-Hop MTU=1400) ←
[클라이언트] → 패킷(MTU=1400, DF=1) → [라우터] → [목적지]
```

---

## 실용적인 활용 정리

### ping으로 할 수 있는 것

```bash
# 기본 연결 확인
ping 8.8.8.8

# 패킷 크기 지정 (MTU 테스트)
ping -s 1472 google.com   # 1472 + 28(헤더) = 1500 MTU

# 특정 횟수만 보내기
ping -c 5 google.com

# 간격 조정
ping -i 0.2 google.com    # 0.2초 간격
```

### traceroute로 할 수 있는 것

```bash
# 어느 구간에서 지연이 발생하는지 확인
traceroute -n google.com   # DNS 역조회 없이 IP만 출력 (빠름)

# 방화벽 때문에 UDP가 막힌 경우 ICMP 사용
traceroute -I google.com
```

### 네트워크 문제 진단 순서

1. `ping 127.0.0.1` → 로컬 루프백 확인 (TCP/IP 스택 자체 검증)
2. `ping 192.168.1.1` → 게이트웨이 연결 확인
3. `ping 8.8.8.8` → 인터넷 연결 확인 (IP 수준)
4. `ping google.com` → DNS 해석 확인
5. `traceroute google.com` → 경로상 문제 구간 파악
