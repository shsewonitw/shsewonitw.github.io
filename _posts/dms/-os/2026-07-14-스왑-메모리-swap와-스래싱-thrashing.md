---
layout: post
title: "[Daily morning study] 스왑 메모리(Swap)와 스래싱(Thrashing)"
description: >
  #daily morning study
category: 
    - dms
    - dms-os
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 스왑 메모리(Swap)란?

물리 RAM이 부족할 때 OS가 디스크의 일부 공간을 마치 RAM처럼 활용하는 기법이다. Linux에서는 swap 파티션 또는 swap 파일 형태로 존재한다.

프로세스가 더 많은 메모리를 요구할 때, OS의 메모리 관리자는 현재 사용되지 않는 페이지(page)를 디스크의 스왑 공간으로 내보내고(swap out), 나중에 필요할 때 다시 RAM으로 불러온다(swap in).

**스왑 작동 순서:**

1. 프로세스가 특정 페이지를 요청
2. 해당 페이지가 RAM에 없음 → 페이지 폴트(Page Fault) 발생
3. OS가 디스크 스왑 영역에서 해당 페이지를 찾아 RAM으로 로드 (swap in)
4. RAM이 가득 찬 경우, 덜 사용되는 페이지를 디스크로 이동 (swap out)

## 스왑의 장단점

| 구분 | 내용 |
|------|------|
| 장점 | RAM보다 넓은 가상 메모리 주소 공간 제공 가능 |
| 장점 | 메모리 부족 시 프로세스 강제 종료(OOM Kill) 방지 |
| 단점 | 디스크 I/O 발생 → RAM 대비 수백~수천 배 느림 |
| 단점 | 잦은 swap in/out → 스래싱(Thrashing) 유발 위험 |

접근 속도 비교: RAM(ns 단위) → SSD(μs 단위) → HDD(ms 단위). 스왑을 많이 쓸수록 체감 성능이 급격히 떨어지는 이유다.

## 스래싱(Thrashing)이란?

프로세스들이 실제 유용한 작업보다 **페이지 교체(swap in/out)에 더 많은 시간을 소비하는 상태**를 말한다.

**발생 원인:**

동시에 실행 중인 프로세스 수가 너무 많아 각 프로세스에 할당되는 실제 프레임(frame) 수가 부족해진다. 각 프로세스의 **워킹 셋(Working Set)** — 현재 활발히 참조 중인 페이지 집합 — 의 합계가 가용 RAM을 초과하면 스래싱이 시작된다.

```
총 워킹 셋 크기 > 물리 RAM 크기
    → 페이지 폴트 연쇄 발생
    → CPU 이용률 급감, 디스크 I/O 폭증
```

**스래싱 악순환 구조:**

```
멀티프로세스 수 증가
    ↓
각 프로세스 할당 프레임 수 감소
    ↓
페이지 폴트 빈도 급증
    ↓
CPU는 페이지 교체만 처리 (실작업 X)
    ↓
CPU 이용률 급락
    ↓
OS가 "CPU 여유 있다" 판단 → 추가 프로세스 실행
    ↓
스래싱 심화 (악순환 반복)
```

## 스래싱 해결 방법

### 1. 워킹 셋 모델 (Working Set Model)

각 프로세스마다 최근 일정 시간 동안 참조한 페이지 집합(워킹 셋)을 추적한다. 워킹 셋 전체를 수용할 수 있는 프레임이 없으면 해당 프로세스를 일시 중단(suspend)시켜 다른 프로세스에게 프레임을 양보하게 만든다.

### 2. 페이지 폴트 빈도 제어 (Page Fault Frequency, PFF)

- PFF가 상한선 초과 → 해당 프로세스에 프레임 추가 할당
- PFF가 하한선 미만 → 프레임 회수
- 여유 프레임이 없으면 일부 프로세스를 스왑 아웃해 공간 확보

### 3. 멀티프로그래밍 정도 제한

동시 실행 프로세스 수를 물리 메모리 크기에 맞춰 제한한다. 프로세스를 무한정 늘리는 것보다 적절한 수를 유지하는 편이 전체 처리량(throughput)에 유리하다.

### 4. 메모리 증설 또는 프로세스 종료

근본적인 해결책이다. RAM을 추가하거나, OOM Killer를 통해 불필요한 프로세스를 종료해 여유 공간을 확보한다.

## Swap 관련 Linux 명령어

```bash
# 현재 스왑 사용 현황 확인
free -h
swapon --show

# 스왑 파일 생성 (1GB 예시)
sudo fallocate -l 1G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# swappiness 현재 값 확인 (0~100)
cat /proc/sys/vm/swappiness

# swappiness 임시 변경 (재부팅 시 초기화)
sudo sysctl vm.swappiness=10
```

**swappiness** 커널 파라미터는 OS가 스왑을 얼마나 적극적으로 사용할지 결정한다.

| 값 | 의미 |
|----|------|
| 0 | RAM이 거의 꽉 찰 때까지 스왑 최소화 |
| 10 | 데이터베이스 서버 등 성능 민감 환경 권장값 |
| 60 | Ubuntu/Debian 기본값 |
| 100 | 매우 적극적으로 스왑 사용 |

## 핵심 정리

| 개념 | 설명 |
|------|------|
| Swap | RAM 부족 시 디스크를 임시 메모리로 활용하는 기법 |
| Swap Out | RAM의 페이지를 디스크 스왑 영역으로 이동 |
| Swap In | 디스크 스왑 영역의 페이지를 RAM으로 복귀 |
| Page Fault | 요청한 페이지가 RAM에 없을 때 발생하는 인터럽트 |
| 스래싱 | 페이지 교체 과부하로 CPU가 실작업을 못 하는 상태 |
| 워킹 셋 | 프로세스가 현재 활발히 참조하는 페이지 집합 |
| swappiness | 스왑 적극성을 조절하는 Linux 커널 파라미터 |

스왑은 메모리 부족 시 시스템이 버티게 해주는 안전망이지만, 지나치게 의존하면 성능이 급격히 저하된다. 특히 MySQL, PostgreSQL 같은 데이터베이스 서버에서는 swappiness를 낮게 설정하거나 스왑을 비활성화하는 경우가 많다.
