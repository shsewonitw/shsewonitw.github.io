---
layout: post
title: "[Daily morning study] 로드 밸런싱 알고리즘 (Round Robin, Least Connections)"
description: >
  #daily morning study
category: 
    - dms
    - dms-network
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 로드 밸런싱이란?

트래픽을 여러 서버에 분산시켜서 특정 서버에 부하가 집중되지 않도록 하는 기술이다. 로드 밸런서가 클라이언트 요청을 받아서 어떤 서버로 보낼지 결정하는데, 이때 사용하는 규칙이 로드 밸런싱 알고리즘이다.

## 정적 알고리즘

서버 상태를 실시간으로 모니터링하지 않고 미리 정해진 규칙에 따라 트래픽을 분산한다.

### Round Robin (라운드 로빈)

서버 목록을 순서대로 돌아가면서 요청을 분배하는 방식이다.

```
요청 1 → 서버 A
요청 2 → 서버 B
요청 3 → 서버 C
요청 4 → 서버 A  (다시 처음으로)
요청 5 → 서버 B
...
```

가장 단순하고 구현이 쉽다. 모든 서버 스펙이 동일하고 요청 처리 시간이 비슷할 때 효과적이다.

단점은 서버 성능이 다를 때 비효율적이라는 점이다. 느린 서버와 빠른 서버가 같은 비율로 요청을 받으면 느린 서버가 과부하 상태가 된다.

### Weighted Round Robin (가중치 라운드 로빈)

서버마다 가중치(weight)를 부여해서 성능이 높은 서버에 더 많은 요청을 보내는 방식이다.

```
서버 A (weight: 3) → 요청 3개
서버 B (weight: 1) → 요청 1개
서버 C (weight: 2) → 요청 2개

순서: A, A, A, B, C, C, A, A, A, B, C, C ...
```

서버 스펙이 다를 때 유용하다. 고성능 서버에 더 많은 트래픽을 할당할 수 있다.

### IP Hash (IP 해시)

클라이언트 IP 주소를 해싱해서 항상 같은 서버로 연결하는 방식이다.

```
클라이언트 192.168.1.1 → 항상 서버 A
클라이언트 192.168.1.2 → 항상 서버 B
클라이언트 192.168.1.3 → 항상 서버 A
```

세션 유지(Session Persistence, Sticky Session)가 필요한 경우에 쓴다. 예를 들어 로그인 세션을 서버 메모리에 저장하는 방식이라면 같은 클라이언트는 항상 같은 서버로 가야 한다.

단점으로는 특정 IP에서 트래픽이 몰리면 해당 서버에 부하가 집중된다. 또 서버가 추가/제거되면 해시 값이 바뀌어서 기존 세션이 끊길 수 있다.

## 동적 알고리즘

서버의 현재 상태를 실시간으로 파악해서 요청을 분배한다.

### Least Connections (최소 연결)

현재 활성 연결(active connection)이 가장 적은 서버로 요청을 보내는 방식이다.

```
서버 A: 연결 10개
서버 B: 연결 3개  ← 여기로 전송
서버 C: 연결 7개
```

요청 처리 시간이 다양할 때 효과적이다. 빠르게 처리되는 요청이 많은 서버는 연결 수가 적어지므로 자동으로 더 많은 요청을 받게 된다.

라운드 로빈보다 현실적으로 균형 잡힌 분산이 가능하다.

### Weighted Least Connections (가중치 최소 연결)

서버 성능(가중치)을 고려해서 `현재 연결 수 / 가중치` 값이 가장 낮은 서버로 요청을 보낸다.

```
서버 A (weight: 4): 연결 8개 → 8/4 = 2.0
서버 B (weight: 2): 연결 3개 → 3/2 = 1.5  ← 여기로 전송
서버 C (weight: 1): 연결 2개 → 2/1 = 2.0
```

서버 성능도 다르고 요청 처리 시간도 다를 때 가장 정교한 분산을 제공한다.

### Least Response Time (최소 응답 시간)

현재 연결 수와 평균 응답 시간을 함께 고려해서 가장 빠르게 응답하는 서버로 보내는 방식이다.

```
(활성 연결 수 + 평균 응답 시간)이 가장 낮은 서버 선택
```

실제로 가장 성능이 좋은 서버를 선택하지만, 응답 시간 측정 오버헤드가 생긴다.

## 알고리즘 비교

| 알고리즘 | 서버 상태 반영 | 서버 성능 차이 고려 | 세션 유지 | 복잡도 |
|----------|--------------|-------------------|----------|--------|
| Round Robin | X | X | X | 낮음 |
| Weighted Round Robin | X | O | X | 낮음 |
| IP Hash | X | X | O | 낮음 |
| Least Connections | O | X | X | 중간 |
| Weighted Least Connections | O | O | X | 중간 |
| Least Response Time | O | O | X | 높음 |

## 실제 사용 사례

**Nginx**는 기본적으로 Round Robin을 사용하고, `least_conn` 지시어로 Least Connections로 전환할 수 있다.

```nginx
upstream backend {
    least_conn;
    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com;
}
```

**AWS ALB(Application Load Balancer)**는 기본적으로 Round Robin을 사용하고, Least Outstanding Requests(LOR) 알고리즘도 지원한다.

**HAProxy**는 roundrobin, leastconn, source(IP Hash), random 등 다양한 알고리즘을 지원한다.

## 알고리즘 선택 기준

- 서버 스펙 동일 + 요청 처리 시간 유사 → **Round Robin**
- 서버 스펙 다름 → **Weighted Round Robin**
- 세션 유지 필요 → **IP Hash** (또는 Redis 같은 외부 세션 스토리지 도입)
- 요청 처리 시간 편차가 클 때 → **Least Connections**
- 서버 스펙도 다르고 처리 시간 편차도 클 때 → **Weighted Least Connections**

대부분의 현대 서비스에서는 세션을 외부 저장소(Redis)에 보관하기 때문에 IP Hash 없이도 세션 문제가 없고, Least Connections 계열이 가장 실용적인 선택인 경우가 많다.
