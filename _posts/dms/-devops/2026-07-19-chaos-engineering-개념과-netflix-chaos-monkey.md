---
layout: post
title: "[Daily morning study] Chaos Engineering 개념과 Netflix Chaos Monkey"
description: >
  #daily morning study
category: 
    - dms
    - dms-devops
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## Chaos Engineering이란

Chaos Engineering은 시스템의 약점을 사전에 발견하기 위해 **의도적으로 장애를 주입**하는 실험 기법이다. 핵심 아이디어는 "어차피 장애는 발생한다. 예측 불가능한 상황에서 발생하는 것보다, 통제된 환경에서 미리 터뜨리는 게 낫다"는 것이다.

Netflix, Amazon, Google 같은 대규모 분산 시스템을 운영하는 기업에서 시작됐으며, 지금은 DevOps/SRE 실무에서 널리 쓰인다.

---

## 탄생 배경: Netflix의 클라우드 이전

Netflix는 2008년 AWS 마이그레이션 중 데이터베이스 손상으로 3일간 서비스 중단을 경험했다. 이후 "클라우드는 언제든 인스턴스가 죽을 수 있다"는 전제 아래, 시스템이 **개별 컴포넌트 장애에도 살아남을 수 있어야 한다**는 설계 철학을 수립했다.

2010년 Netflix는 이 철학을 실현하기 위해 **Chaos Monkey**를 만들었다.

---

## Netflix Chaos Monkey

Chaos Monkey는 프로덕션 환경에서 **무작위로 서버 인스턴스를 강제 종료**시키는 도구다. 이름 그대로 시스템 안에 원숭이를 풀어놓고 아무 서버나 꺼버리는 방식이다.

### 동작 방식

```
[Chaos Monkey]
  ↓
Auto Scaling Group에서 무작위 EC2 인스턴스 선택
  ↓
인스턴스 강제 종료 (terminate)
  ↓
시스템이 자동 복구되는지 관찰
  ↓
복구 실패 시 → 알람 발생 → 개선 작업
```

평일 업무 시간 중에만 동작하도록 설정한다. 그래야 엔지니어들이 즉시 대응할 수 있기 때문이다.

### Simian Army로 확장

Netflix는 Chaos Monkey 이후 더 다양한 실험 도구를 만들어 **Simian Army**라는 패키지로 공개했다.

| 도구 | 역할 |
| --- | --- |
| Chaos Monkey | 무작위 인스턴스 종료 |
| Chaos Gorilla | 가용 영역(AZ) 전체 차단 |
| Chaos Kong | 리전 전체 장애 시뮬레이션 |
| Latency Monkey | 네트워크 지연 주입 |
| Conformity Monkey | 베스트 프랙티스 위반 인스턴스 탐지 |
| Doctor Monkey | 상태 불량 인스턴스 탐지 및 제거 |
| Janitor Monkey | 미사용 리소스 정리 |

---

## Chaos Engineering의 4단계 프로세스

### 1. 정상 상태(Steady State) 정의

먼저 시스템이 정상적으로 동작할 때의 상태를 측정 가능한 지표로 정의한다.

```
- 초당 요청 처리량 (RPS): 10,000
- 에러율: < 0.1%
- P99 응답 시간: < 200ms
- 서비스 가용성: 99.99%
```

### 2. 가설 수립

"이 인스턴스 하나가 죽어도 정상 상태가 유지될 것이다"처럼 구체적인 가설을 세운다.

```
가설: 결제 서비스 인스턴스 하나를 종료해도
     전체 트랜잭션 성공률은 99.9% 이상을 유지한다.
```

### 3. 실험 설계 및 실행

장애를 주입할 범위, 방법, 롤백 조건을 미리 정의한다.

```yaml
# 실험 설정 예시
experiment:
  name: "payment-service-instance-kill"
  target: "payment-service"
  action: "terminate-instance"
  blast_radius: "1 instance"
  duration: "10 minutes"
  rollback_condition: "error_rate > 5%"
  observation_metrics:
    - error_rate
    - transaction_success_rate
    - p99_latency
```

### 4. 결과 분석 및 개선

실험 후 정상 상태와 비교해서 편차가 생겼다면, 그게 바로 약점이다. 해당 약점을 개선한 뒤 다시 실험한다.

---

## 장애 주입 유형

Chaos Engineering에서 주입하는 장애는 크게 4가지로 나뉜다.

### 1. 인프라 레벨

```
- 서버 인스턴스 강제 종료
- 디스크 꽉 채우기 (disk full)
- CPU 100% 점유
- 메모리 누수 시뮬레이션
```

### 2. 네트워크 레벨

```
- 패킷 손실 (packet loss)
- 네트워크 지연 추가 (latency injection)
- 대역폭 제한 (bandwidth throttling)
- 특정 포트 차단
```

### 3. 애플리케이션 레벨

```
- 의존하는 외부 서비스 타임아웃
- 특정 API 강제 에러 반환
- 데이터베이스 연결 끊기
- 캐시 서버 장애
```

### 4. DNS/서비스 디스커버리 레벨

```
- DNS 응답 지연
- 잘못된 IP 반환
- 서비스 레지스트리 장애
```

---

## 실제 도구들

### LitmusChaos (CNCF 오픈소스)

Kubernetes 환경에서 가장 널리 쓰이는 Chaos Engineering 플랫폼이다.

```yaml
# Pod 삭제 실험 예시
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: nginx-chaos
spec:
  appinfo:
    appns: 'default'
    applabel: 'app=nginx'
    appkind: 'deployment'
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: '30'
            - name: CHAOS_INTERVAL
              value: '10'
            - name: FORCE
              value: 'false'
```

### Chaos Mesh

CNCF Sandbox 프로젝트. Kubernetes 네이티브로 설계되었고 시각적인 대시보드를 제공한다.

```yaml
# 네트워크 지연 실험 예시
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-delay
spec:
  action: delay
  mode: one
  selector:
    namespaces:
      - default
    labelSelectors:
      'app': 'payment-service'
  delay:
    latency: "100ms"
    correlation: "25"
    jitter: "10ms"
  duration: "300s"
```

### Gremlin

엔터프라이즈용 SaaS 플랫폼. GUI 기반으로 쉽게 실험을 설계할 수 있다.

---

## Chaos Engineering vs 기존 테스팅

| 구분 | 기존 테스팅 | Chaos Engineering |
| --- | --- | --- |
| 환경 | 개발/테스트 환경 | 프로덕션 환경 |
| 시나리오 | 예상 가능한 케이스 | 예상하지 못한 장애 |
| 목적 | 버그 발견 | 시스템 약점 발견 |
| 타이밍 | 배포 전 | 지속적으로 |
| 결과 | Pass/Fail | 개선점 도출 |

기존 테스팅이 "이 기능이 올바르게 동작하는가?"를 검증한다면, Chaos Engineering은 "이 시스템이 장애 상황에서도 살아남는가?"를 검증한다.

---

## 게임데이(Game Day)

Chaos Engineering에서 **Game Day**는 팀 전체가 모여 대규모 장애를 시뮬레이션하는 이벤트다. 실제 장애 상황을 재연하면서 팀의 대응 절차, 커뮤니케이션, 복구 능력을 종합적으로 테스트한다.

```
Game Day 진행 순서:

1. 시나리오 정의
   - "결제 서비스와 DB 동시 장애 발생"

2. 참여 팀 구성
   - 실험 팀 (장애 주입)
   - 대응 팀 (온콜 엔지니어)
   - 관찰 팀 (메트릭 모니터링)

3. 실험 실행
   - 실험 팀이 장애 주입
   - 대응 팀은 사전 통지 없이 대응

4. 회고 (Post-mortem)
   - 무엇이 잘 됐는가
   - 무엇이 예상과 달랐는가
   - 개선 사항
```

---

## 도입 시 주의사항

### 프로덕션에서 시작하지 말 것

처음에는 스테이징 환경에서 시작하고, 시스템의 복원력이 어느 정도 검증된 뒤 프로덕션으로 확대한다.

### Blast Radius 최소화

처음에는 영향 범위를 좁게 설정한다. 전체 클러스터가 아니라 인스턴스 하나, 사용자의 1%처럼 시작한다.

### 명확한 롤백 조건

에러율이 X%를 넘으면 즉시 실험을 중단하는 조건을 반드시 사전에 정의해야 한다.

### 모니터링 없이 실험 금지

실험 중 메트릭을 실시간으로 볼 수 있어야 한다. 모니터링 체계가 갖춰지지 않은 상태에서는 카오스 실험을 진행하면 안 된다.

---

## 정리

Chaos Engineering은 "장애는 피하는 게 아니라, 다루는 법을 익히는 것"이라는 관점의 전환이다. Netflix가 초당 수백만 요청을 처리하면서도 높은 가용성을 유지할 수 있는 이유 중 하나가 바로 이 Chaos Engineering 문화다.

핵심은 실험 → 관찰 → 개선의 사이클을 지속적으로 반복하는 것이다. 단순히 서버를 끄는 게 아니라, 시스템이 어떤 조건에서 무너지는지를 이해하고, 그 약점을 제거해 나가는 과정이다.
