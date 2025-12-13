---
layout: post
title: "[Daily morning study] Mutex(뮤텍스)와 Semaphore(세마포어)의 차이"
description: >
  #daily morning study
category: 
    - dms
    - -os
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---
# Mutex(뮤텍스)와 Semaphore(세마포어)의 차이

멀티스레드 환경에서 동기화는 중요한 문제입니다. 이 두 가지 개념인 뮤텍스와 세마포어는 서로 다른 방식으로 스레드 간의 자원 접근을 조절하는 역할을 합니다. 이번 학습에서는 뮤텍스와 세마포어의 정의, 사용 용도, 그리고 차이점에 대해 자세히 살펴보겠습니다.

## 1. 뮤텍스 (Mutex)

### 정의
뮤텍스는 "Mutual Exclusion"의 줄임말로, 공유 자원에 대한 동시 접근을 제어하기 위해 사용됩니다. 뮤텍스는 하나의 스레드만 자원에 접근할 수 있도록 보장하는 객체입니다.

### 사용 용도
- 데이터베이스, 파일, 객체 등 공유 자원에 대한 접근 제한.
- 스레드가 수행하는 작업이 완료될 때까지 다른 스레드가 대기하도록 하기.

### 코드 예시
```c
#include <stdio.h>
#include <pthread.h>

pthread_mutex_t lock;

void* thread_function(void* arg) {
    pthread_mutex_lock(&lock);
    
    // 공유 자원에 대한 작업
    printf("Thread %ld is working.\n", (long)arg);
    
    pthread_mutex_unlock(&lock);
    return NULL;
}

int main() {
    pthread_t threads[5];
    pthread_mutex_init(&lock, NULL);

    for (long i = 0; i < 5; i++) {
        pthread_create(&threads[i], NULL, thread_function, (void*)i);
    }
    
    for (int i = 0; i < 5; i++) {
        pthread_join(threads[i], NULL);
    }

    pthread_mutex_destroy(&lock);
    return 0;
}
```

## 2. 세마포어 (Semaphore)

### 정의
세마포어는 특정 자원을 사용할 수 있는 스레드의 수를 제한하는 카운터입니다. 일반적으로 두 가지 유형인 이진 세마포어(값이 0 또는 1)와 카운팅 세마포어(여러 개의 리소스 관리를 위한 카운터)가 있습니다.

### 사용 용도
- 다수의 스레드가 동일한 리소스를 동시에 사용할 수 있는 상황에서 자원 할당 관리.
- 프로세스 간 통신을 위한 신호 체계.

### 코드 예시
```c
#include <stdio.h>
#include <pthread.h>
#include <semaphore.h>

sem_t semaphore;

void* thread_function(void* arg) {
    sem_wait(&semaphore);
    
    // 공유 자원에 대한 작업
    printf("Thread %ld is working.\n", (long)arg);
    
    sem_post(&semaphore);
    return NULL;
}

int main() {
    pthread_t threads[10];
    sem_init(&semaphore, 0, 3); // 최대 3개의 스레드 동시 접근 가능

    for (long i = 0; i < 10; i++) {
        pthread_create(&threads[i], NULL, thread_function, (void*)i);
    }
    
    for (int i = 0; i < 10; i++) {
        pthread_join(threads[i], NULL);
    }

    sem_destroy(&semaphore);
    return 0;
}
```

## 3. 뮤텍스와 세마포어의 차이점

|                  | 뮤텍스 (Mutex)                              | 세마포어 (Semaphore)                          |
|------------------|-------------------------------------------|---------------------------------------------|
| 동기화 방식         | 상호 배제 (Mutual Exclusion)                   | 카운팅을 통한 자원 제어                       |
| 소유권              | 스레드가 잠근 자원만 해제 가능                 | 여러 스레드가 접근 가능, 소유권 없음             |
| 용도                | 단일 자원에 대한 동시 접근 차단                | 여러 개의 자원 관리 및 동시 접근 허용             |
| 상태               | 잠금 및 잠금 해제 상태만 사용                  | 카운트 기반으로 리소스 관리                     |

## 결론

뮤텍스와 세마포어 모두 멀티스레드 프로그래밍에서 중요한 동기화 도구입니다. 뮤텍스는 단일 스레드의 자원 접근을 제어하고, 세마포어는 여러 스레드의 동시 접근을 허용합니다. 이러한 차이점을 이해하면, 더 나은 멀티스레드 프로그램을 설계하고 제조할 수 있습니다.

학습한 내용을 바탕으로 실제 프로젝트에 적용해보며 실습해보는 것이 좋습니다. 각 개념의 사용 사례를 떠올리며 코드 작성을 해보는 것도 좋은 학습 방법이 될 것입니다.
