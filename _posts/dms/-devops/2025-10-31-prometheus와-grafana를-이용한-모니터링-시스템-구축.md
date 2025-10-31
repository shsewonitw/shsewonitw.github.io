---
layout: post
title: "[Daily morning study] Prometheus와 Grafana를 이용한 모니터링 시스템 구축"
description: >
  #daily morning study
category: 
    - dms
    - -devops
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---
# Prometheus와 Grafana를 이용한 모니터링 시스템 구축

모니터링 시스템은 애플리케이션 성능이나 시스템의 상태를 추적하고, 문제를 사전 예방하는 데 매우 중요하다. 이 가이드에서는 Prometheus와 Grafana를 사용하여 효과적인 모니터링 시스템을 구축하는 방법을 설명하겠다.

## 1. Prometheus 소개

Prometheus는 오픈 소스 시스템 모니터링 및 경고 툴로, 주로 시계열 데이터(Time Series Data)를 수집하고 저장하는 데 사용된다. 주요 특징은 다음과 같다.

- **Pull 모델**: Prometheus는 설정된 주기로 메트릭스를 요청하여 수집함.
- **다양한 데이터 소스**: 다양한 언어 및 플랫폼으로 작성된 애플리케이션과 통해 연동 가능.
- **고급 쿼리 언어**: PromQL(Prometheus Query Language)을 사용하여 데이터 분석을 쉽게 수행할 수 있음.

### 1.1 Prometheus 설치

```bash
# Prometheus 다운로드
wget https://github.com/prometheus/prometheus/releases/latest/download/prometheus-<version>.linux-amd64.tar.gz
tar xvf prometheus-<version>.linux-amd64.tar.gz
cd prometheus-<version>.linux-amd64
```

### 1.2 기본 설정

Prometheus의 설정 파일인 `prometheus.yml`을 수정하여 모니터링할 타겟을 정의한다.

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'my_application'
    static_configs:
      - targets: ['localhost:8080']
```

## 2. Grafana 소개

Grafana는 시각화 전문 대시보드 툴로, 다양한 데이터베이스와 쉽게 연동해 데이터를 시각화할 수 있다. Prometheus와의 통합이 잘 되어 있어, 실시간 데이터를 대시보드로 표현하기에 적합하다.

### 2.1 Grafana 설치

```bash
# Grafana 설치
wget https://dl.grafana.com/oss/release/grafana-<version>.linux-amd64.tar.gz
tar -zxvf grafana-<version>.linux-amd64.tar.gz
cd grafana-<version>
```

### 2.2 Grafana 시작

```bash
# Grafana 서버 시작
./bin/grafana-server web
```

기본적으로 `http://localhost:3000`에서 접근할 수 있다. 기본 아이디는 `admin`이며 처음 로그인 시 비밀번호 변경을 요청받는다.

### 2.3 데이터 소스 추가

1. Grafana 대시보드에서 **Configuration** > **Data Sources**를 클릭한다.
2. **Add data source**를 클릭하고, **Prometheus**를 선택한다.
3. URL에 `http://localhost:9090`을 입력하고 **Save & Test**를 클릭하여 연결을 확인한다.

## 3. 대시보드 구성

이제 Grafana에서 Prometheus 데이터를 시각화하는 대시보드를 만들 수 있다.

### 3.1 패널 추가

1. Grafana 대시보드에서 **+** 아이콘을 클릭한 후 **Dashboard**를 선택한다.
2. **Add new panel**을 클릭한다.
3. 아래의 쿼리 예시들을 사용하여 메트릭스를 추가한다.

```promql
# CPU 사용량
rate(cpu_seconds_total[5m])

# 메모리 사용량
memory_usage_bytes

# 요청 수
http_requests_total
```

### 3.2 패널 설정

각 패널에서 시각화 타입(그래프, 테이블, 등)을 선택하고, 패널 제목과 설명을 입력한다. 이후, **Apply**를 클릭하여 대시보드에 패널을 추가한다.

## 4. 모니터링 경고 설정

Prometheus에서 경고 규칙을 설정하여 특정 조건을 만족할 때 경고를 발생시킬 수 있다.

### 4.1 경고 규칙 추가

`prometheus.yml` 파일에 아래와 같은 형태로 경고 규칙을 추가한다.

```yaml
rule_files:
  - 'alert.rules'

# alert.rules 파일 내용
groups:
- name: example
  rules:
  - alert: HighCpuUsage
    expr: rate(cpu_seconds_total[5m]) > 0.8
    for: 5m
    labels:
      severity: page
    annotations:
      summary: "High CPU usage detected"
      description: "CPU usage is above 80% for more than 5 minutes."
```

### 4.2 경고 알림 수신 설정

Alertmanager를 통해 이메일, Slack, PagerDuty 등으로 알림을 받을 수 있도록 설정한다. 

## 5. 결론

Prometheus와 Grafana를 이용하여 강력한 모니터링 시스템을 구축할 수 있다. 시스템의 메트릭을 효과적으로 시각화하고, 경고를 통해 문제를 조기에 발견할 수 있도록 설정하였다. 이 과정을 통해 모니터링 시스템 구축의 기본 메커니즘을 이해할 수 있었을 것이다.

이 문서를 참고하여 자신만의 모니터링 환경을 구축해 보자.
