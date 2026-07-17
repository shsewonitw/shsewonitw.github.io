---
layout: post
title: "[Daily morning study] Chaos Engineering 개념과 실험 설계 원칙"
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

Chaos Engineering은 분산 시스템이 예상치 못한 장애 조건에서도 견딜 수 있는지를 **의도적으로 실험**하여 검증하는 방법론이다. 핵심 아이디어는 간단하다. 실제 장애가 발생하기 전에 먼저 의도적으로 장애를 일으켜서, 시스템의 약점을 찾아내고 복원력(resilience)을 높이는 것이다.

Netflix에서 2010년대 초반 AWS 마이그레이션 과정에서 "우리 시스템이 진짜 장애에도 살아남을 수 있을까?"라는 질문에서 출발했다. 이에 대한 답이 바로 Chaos Monkey였고, 이후 Chaos Engineering이라는 학문적 방법론으로 발전했다.

전통적 접근 방식인 "장애 없이 시스템을 운영하는 것"과는 근본적으로 다르다. Chaos Engineering은 장애를 피하려 하지 않고, **장애를 미리 경험하고 학습**하는 방식을 택한다.

---

## 왜 필요한가

현대 소프트웨어 시스템은 수백~수천 개의 마이크로서비스, 클라우드 인프라, 외부 API 의존성으로 구성된다. 이런 복잡한 환경에서는 이론적인 테스트만으로는 실제 운영 장애를 완전히 예측할 수 없다.

일반적인 테스트 방식의 한계:

- **유닛 테스트 / 통합 테스트**: 코드 로직의 정확성 검증. 네트워크 지연, 부분 장애, 리소스 소진 등은 재현하기 어렵다.
- **로드 테스트**: 트래픽 급증은 시뮬레이션하지만 인프라 장애는 다루지 않는다.
- **DR(Disaster Recovery) 훈련**: 전체 장애 시나리오를 다루지만 빈도가 낮고 실제 운영 환경과 차이가 생긴다.

Chaos Engineering은 이런 공백을 채운다. 실제 운영 환경(또는 최대한 유사한 환경)에서 통제된 방식으로 장애를 주입하고, 시스템이 어떻게 반응하는지 관찰한다.

---

## Chaos Engineering의 핵심 원칙

Netflix가 2016년 발표한 "Principles of Chaos Engineering"을 기반으로 한다.

### 1. 정상 상태(Steady State) 정의

실험 전에 시스템이 정상적으로 동작하는 상태를 정량적으로 정의해야 한다. 단순히 "서버가 살아있다"가 아니라, 측정 가능한 지표로 표현한다.

```
정상 상태 예시:
- API 응답 시간 p99 < 200ms
- 초당 처리 요청 수 > 1,000 RPS
- 에러율 < 0.1%
- 결제 성공률 > 99.5%
```

정상 상태 없이는 실험 결과를 해석할 수 없다. "이 실험이 문제를 일으켰는가?"를 판단하는 기준이다.

### 2. 현실적인 장애 시나리오 설계

"이런 일이 실제로 일어날 수 있는가?"를 기준으로 시나리오를 선택한다.

현실적인 시나리오:
- 특정 가용 영역(AZ) 전체 다운
- 의존하는 외부 서비스의 응답 지연 (5초)
- 네트워크 패킷 손실 (30%)
- 데이터베이스 커넥션 풀 소진
- 디스크 용량 90% 초과
- 특정 Pod의 OOM Kill

현실적이지 않은 시나리오 (초기에는 피함):
- 전체 데이터센터 동시 다운
- 데이터베이스 완전 삭제

### 3. 운영 환경에서 실험

이상적으로는 실제 운영 환경에서 실험한다. Staging 환경은 운영 환경의 트래픽 패턴, 데이터 크기, 서비스 의존성을 완전히 재현하지 못한다.

```
실험 환경 선택 기준:

Staging 우선 상황:
  → 처음 Chaos Engineering을 도입할 때
  → 검증되지 않은 새 실험 유형
  → 데이터 손상 위험이 있는 실험

운영 환경 직접 상황:
  → Staging에서 충분히 검증된 실험
  → 트래픽 기반 검증이 필요한 실험
  → 실제 사용자 행동이 필요한 시나리오
```

운영에서 실험할 때는 영향을 최소화하기 위해 트래픽의 일부만 대상으로 시작한다.

### 4. 자동화와 지속적 실험

일회성 이벤트가 아니라 CI/CD 파이프라인에 통합되어야 한다. 시스템은 계속 변하기 때문에, 특정 시점에 안전했다고 해서 이후에도 안전하다는 보장이 없다.

### 5. 폭발 반경(Blast Radius) 최소화

실험은 가능한 한 작은 범위에서 시작하고, 안전하다고 확인되면 점차 범위를 넓힌다.

```
확대 순서 예시:
단일 인스턴스 → 특정 AZ → 전체 리전
소수 사용자 트래픽 → 10% → 50% → 전체
```

---

## 실험 설계 방법

체계적인 실험 설계는 Chaos Engineering의 핵심이다. 즉흥적으로 장애를 일으키는 것은 Chaos Engineering이 아니다.

### GameDay 방식

GameDay는 팀 전체가 참여하는 정기적인 장애 대응 훈련이다.

```
GameDay 진행 구조:

1. 준비 단계 (1~2주 전)
   - 실험 시나리오 선정 및 문서화
   - 정상 상태 지표 기준값 수집
   - 롤백 계획 수립
   - 모니터링 대시보드 준비

2. 실험 당일
   - 시나리오 설명 및 목표 공유
   - 장애 주입 실행
   - 실시간 관찰 및 기록
   - 필요 시 즉시 롤백

3. 사후 분석
   - 발견된 취약점 정리
   - 액션 아이템 도출
   - 다음 GameDay 주제 선정
```

### 실험 문서 템플릿

```yaml
실험명: 결제 서비스 의존성 DB 지연 테스트

가설:
  DB 응답이 3초 지연되더라도,
  결제 서비스는 타임아웃을 정상적으로 처리하고
  에러율이 1% 미만을 유지할 것이다.

정상 상태:
  - 결제 API 에러율 < 0.1%
  - p99 응답 시간 < 500ms

실험 방법:
  - 결제 DB에 3초 지연 주입 (tc netem 활용)
  - 지속 시간: 10분
  - 대상: 전체 트래픽의 5%

관찰 지표:
  - 결제 에러율
  - 결제 응답 시간 분포
  - 서킷 브레이커 상태
  - 재시도 횟수

롤백 조건:
  - 에러율 5% 초과 시 즉시 중단
  - 이상 징후 발생 시 runbook 참고

결과:
  (실험 후 작성)
```

---

## 주요 Chaos Engineering 도구

### Chaos Monkey (Netflix)

Netflix OSS의 원조 Chaos 도구. Kubernetes 클러스터에서 랜덤하게 Pod를 종료한다. 설정이 간단하고 Spinnaker와 통합된다.

```yaml
# Chaos Monkey 설정 예시
chaosMonkey:
  enabled: true
  meanTimeBetweenKillsInWorkDays: 1
  minTimeBetweenKillsInWorkDays: 1
  grouping: APP
  regionsAreIndependent: true
```

### LitmusChaos

CNCF(Cloud Native Computing Foundation) 산하 오픈소스 Chaos Engineering 플랫폼. Kubernetes 네이티브이며 ChaosExperiment라는 CRD(Custom Resource Definition)로 실험을 정의한다.

```yaml
# LitmusChaos 실험 예시: Pod 삭제
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: payment-pod-delete
spec:
  appinfo:
    appns: production
    applabel: "app=payment-service"
  chaosServiceAccount: litmus-admin
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: "60"       # 60초간
            - name: CHAOS_INTERVAL
              value: "10"       # 10초마다 삭제
            - name: FORCE
              value: "false"
```

### Gremlin

상용 Chaos Engineering SaaS 플랫폼. 다양한 장애 유형을 UI로 쉽게 조작할 수 있다.

지원하는 장애 유형:

| 카테고리 | 장애 유형 |
|---------|----------|
| 상태    | 프로세스 종료, 타임 점프 |
| 네트워크 | 지연, 패킷 손실, 블랙홀, DNS 장애 |
| 리소스   | CPU 과부하, 메모리 소진, 디스크 가득 참 |
| 상태    | 컨테이너 종료, AZ 장애 시뮬레이션 |

### tc (Traffic Control)

Linux 커널 내장 네트워크 제어 도구. 별도 설치 없이 네트워크 레벨 장애를 주입할 수 있다.

```bash
# 네트워크 지연 100ms 추가
tc qdisc add dev eth0 root netem delay 100ms

# 패킷 손실 10% 주입
tc qdisc add dev eth0 root netem loss 10%

# 지연 + 지터(jitter) 추가
tc qdisc add dev eth0 root netem delay 100ms 20ms

# 원복
tc qdisc del dev eth0 root
```

---

## 실제 적용 사례

### Netflix: Chaos Monkey → Simian Army

Netflix는 AWS 마이그레이션 중 특정 EC2 인스턴스가 예고 없이 종료되더라도 서비스가 계속 동작하는지 검증하기 위해 Chaos Monkey를 만들었다. 이후 더 다양한 장애 시나리오를 처리하기 위해 Simian Army로 확장했다.

```
Simian Army 구성:
- Chaos Monkey     : 랜덤 인스턴스 종료
- Chaos Gorilla    : 전체 AWS 가용 영역 시뮬레이션 다운
- Chaos Kong       : 전체 리전 장애 시뮬레이션
- Latency Monkey   : 네트워크 지연 주입
- Conformity Monkey: 모범 사례 미준수 인스턴스 감지
- Security Monkey  : 보안 설정 이상 탐지
```

### 결과

Netflix는 Chaos Engineering 도입 이후, 실제 장애 발생 시 MTTR(Mean Time To Recovery)이 크게 단축됐다. 이미 비슷한 상황을 실험을 통해 경험했기 때문에 대응 방법이 문서화되어 있고, 팀이 패닉 없이 대응할 수 있다.

---

## 주의사항과 도입 시 고려점

### 비즈니스 팀과의 커뮤니케이션

Chaos Engineering은 기술 팀만의 활동이 아니다. 실험이 실제 사용자 경험에 영향을 줄 수 있기 때문에 비즈니스 팀과 사전에 합의가 필요하다.

```
실험 전 체크리스트:
□ 실험 시간대 선정 (트래픽이 낮은 시간 우선)
□ 이해관계자 사전 공지
□ 고객 지원팀 사전 알림
□ 롤백 계획 및 담당자 지정
□ 모니터링 알람 임계값 조정
□ 실험 중단 조건(kill switch) 명확히 정의
```

### 성숙도 모델

Chaos Engineering 도입은 단계적으로 진행한다.

```
Level 1: 준비
  - 모니터링 및 관찰 가능성 구축
  - 정상 상태 지표 정의
  - 수동으로 GameDay 진행

Level 2: 실험
  - 도구 도입 (LitmusChaos 등)
  - Staging 환경에서 자동화 실험
  - 실험 결과 문서화

Level 3: 고도화
  - 운영 환경 실험
  - CI/CD 파이프라인 통합
  - 실험 라이브러리 축적

Level 4: 문화
  - 개발자가 직접 실험 설계
  - 지속적 자동 실험 운영
  - 결과 기반 아키텍처 개선
```

---

## 정리

- Chaos Engineering은 장애를 의도적으로 일으켜 시스템의 복원력을 검증하는 방법론이다
- 핵심은 "정상 상태 정의 → 현실적 시나리오 → 운영 환경 실험 → 지속적 자동화"의 원칙을 따르는 것이다
- 실험은 폭발 반경을 최소화하면서 단계적으로 확대한다
- LitmusChaos, Chaos Monkey, Gremlin 등 다양한 도구가 있으며 환경에 맞게 선택한다
- 기술적 도구보다 팀 문화와 사전 커뮤니케이션이 더 중요하다
- 처음부터 운영 환경 전체를 대상으로 하지 않고, Staging → 소규모 운영 → 전체 운영 순으로 점진적으로 확대한다
