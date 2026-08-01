---
layout: post
title: "[Daily morning study] TCP 혼잡 제어 알고리즘 (Congestion Control)"
description: >
  #daily morning study
category: 
    - dms
    - dms-network
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## TCP 혼잡 제어란

TCP는 신뢰성 있는 전송을 보장하기 위해 흐름 제어(Flow Control)뿐만 아니라 **혼잡 제어(Congestion Control)**도 수행한다.

- **흐름 제어**: 수신 측의 처리 능력에 맞춰 송신 속도를 조절 (수신 버퍼 기반)
- **혼잡 제어**: 네트워크 자체(라우터, 링크)의 혼잡 상황에 맞춰 송신 속도를 조절

네트워크가 혼잡해지면 라우터 큐에 패킷이 쌓이고, 결국 패킷이 드롭된다. TCP 송신 측은 이 패킷 손실을 혼잡의 신호로 인식하고 전송 속도를 줄인다.

---

## 핵심 개념: 혼잡 윈도우 (cwnd)

TCP 송신 측은 두 가지 윈도우를 관리한다.

| 윈도우 | 설명 |
|--------|------|
| `rwnd` (Receive Window) | 수신 측이 허용하는 버퍼 크기 |
| `cwnd` (Congestion Window) | 네트워크 혼잡 상태에 따라 송신 측이 자체적으로 제한하는 크기 |

실제 송신 가능한 데이터 양은 `min(rwnd, cwnd)`.

`ssthresh` (Slow Start Threshold)는 Slow Start와 혼잡 회피를 전환하는 임계값이다. 보통 초기값은 65535 바이트로 설정된다.

---

## 혼잡 제어 4단계

### 1. Slow Start (느린 시작)

- 연결 초기 또는 타임아웃 후에 진입
- cwnd를 1 MSS(Maximum Segment Size)부터 시작해 **ACK 수신마다 1 MSS씩 증가**
- 즉, RTT마다 cwnd가 **두 배**로 지수 증가
- cwnd가 ssthresh에 도달하면 혼잡 회피 단계로 전환

```
초기: cwnd = 1 MSS
1 RTT 후: cwnd = 2 MSS
2 RTT 후: cwnd = 4 MSS
3 RTT 후: cwnd = 8 MSS
...
```

### 2. Congestion Avoidance (혼잡 회피)

- cwnd >= ssthresh 상태
- **RTT마다 cwnd를 1 MSS씩 선형 증가** (ACK마다 1/cwnd만큼 증가)
- 손실이 발생할 때까지 천천히 대역폭을 탐색

```
RTT마다: cwnd += 1 MSS
```

### 3. Fast Retransmit (빠른 재전송)

- **3개의 중복 ACK(Duplicate ACK)**를 수신하면 타임아웃을 기다리지 않고 즉시 해당 패킷 재전송
- 패킷 손실이 발생했지만 후속 패킷들은 도착하고 있다는 의미 → 네트워크는 살아있음

```
수신 측이 seq=100을 못 받으면:
- seq=101 받음 → ACK 100
- seq=102 받음 → ACK 100
- seq=103 받음 → ACK 100 (3 Dup ACK)
→ 송신 측이 seq=100을 즉시 재전송
```

### 4. Fast Recovery (빠른 복구)

- 3 Dup ACK로 인한 Fast Retransmit 이후 동작
- ssthresh = cwnd / 2 로 줄임
- cwnd = ssthresh + 3 MSS 로 설정 (이미 3개의 패킷이 버퍼링됨)
- 추가 Dup ACK마다 cwnd += 1 MSS (네트워크에 새 패킷 투입 가능)
- 새 ACK를 받으면 cwnd = ssthresh 로 설정하고 혼잡 회피로 전환

---

## Reno vs Tahoe

| 구분 | TCP Tahoe | TCP Reno |
|------|-----------|----------|
| 3 Dup ACK 시 | cwnd = 1, Slow Start 재시작 | Fast Recovery 진입 |
| Timeout 시 | cwnd = 1, Slow Start 재시작 | cwnd = 1, Slow Start 재시작 |
| 특징 | 단순하지만 성능 저하 큼 | Fast Recovery로 성능 개선 |

Reno는 3 Dup ACK와 타임아웃을 다르게 처리한다. 타임아웃은 더 심각한 혼잡을 의미하므로 Slow Start로 돌아간다.

---

## 혼잡 감지 방식: 손실 기반 vs 지연 기반

### 손실 기반 (Loss-Based)

- 전통적인 방식 (Tahoe, Reno, CUBIC)
- **패킷 손실**을 혼잡 신호로 사용
- 실제로 혼잡이 발생해 패킷이 드롭된 후에야 반응 → **반응이 늦음**
- 버퍼가 꽉 차기 전까지 전송 속도를 계속 올림 → Bufferbloat 문제 유발

### 지연 기반 (Delay-Based)

- BBR(Bottleneck Bandwidth and RTT)이 대표적
- **RTT 증가**를 혼잡 예측 신호로 사용
- 손실 없이 네트워크 상태를 사전에 파악해 속도 조절
- 구글이 2016년 개발, YouTube/Gmail 등에 적용

---

## CUBIC (Linux 기본 알고리즘)

Linux 커널 기본 혼잡 제어 알고리즘이다.

- 혼잡이 발생한 시점의 cwnd = `W_max`로 기억
- cwnd 증가를 **3차 함수(cubic function)** 기반으로 계산
- `W_max`에 가까울수록 증가를 늦추고, 멀면 빠르게 증가
- 고속 네트워크에서 Reno보다 훨씬 빠르게 대역폭 회복

```
cwnd(t) = C(t - K)³ + W_max
K = 마지막 혼잡 이후 W_max에 다시 도달하는 데 걸리는 시간
```

---

## 혼잡 제어 흐름 요약

```
연결 시작
   │
   ▼
[Slow Start]
 cwnd *= 2 (RTT마다)
   │
   ├── cwnd >= ssthresh → [Congestion Avoidance]
   │                        cwnd += 1 (RTT마다)
   │                            │
   │      ┌─────────────────────┤
   │      ▼                     ▼
   │   Timeout               3 Dup ACK
   │      │                     │
   │  ssthresh = cwnd/2     ssthresh = cwnd/2
   │  cwnd = 1              cwnd = ssthresh + 3
   │      │                     │
   │  [Slow Start]         [Fast Recovery]
   └──────────────────────────→ 새 ACK 수신 시 cwnd = ssthresh
                                → [Congestion Avoidance]
```

---

## 실무에서 중요한 이유

- **서버 간 대용량 파일 전송** 성능이 혼잡 제어 알고리즘에 크게 영향받음
- `sysctl net.ipv4.tcp_congestion_control` 로 Linux에서 알고리즘 확인/변경 가능
- 고대역폭-고지연(High BDP) 환경에서는 CUBIC이나 BBR이 Reno보다 훨씬 유리
- **RTT가 크면** Slow Start 단계에서 시간이 많이 걸려 대역폭 활용이 비효율적

```bash
# Linux에서 현재 혼잡 제어 알고리즘 확인
cat /proc/sys/net/ipv4/tcp_congestion_control

# 사용 가능한 알고리즘 목록
cat /proc/sys/net/ipv4/tcp_available_congestion_control

# BBR로 변경
echo "bbr" > /proc/sys/net/ipv4/tcp_congestion_control
```
