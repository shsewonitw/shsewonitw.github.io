---
layout: post
title: "[Daily morning study] Knowledge Distillation (지식 증류) — 대형 모델을 소형 모델로 압축하는 기법"
description: >
  #daily morning study
category: 
    - dms
    - dms-ai
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## Knowledge Distillation이란

Knowledge Distillation(지식 증류)은 크고 복잡한 모델(Teacher)이 학습한 지식을 작고 가벼운 모델(Student)에게 전달하는 모델 경량화 기법이다. 2015년 Hinton 등이 제안했으며, 모바일 기기나 엣지 환경처럼 자원이 제한된 환경에서 성능 저하를 최소화하면서 모델을 배포하는 데 활용된다.

단순히 Teacher의 정답 레이블만 보고 학습하는 것이 아니라, Teacher가 내놓는 **Soft Label(확률 분포)** 자체를 학습 신호로 사용하는 게 핵심이다.

---

## Soft Label이 왜 중요한가

일반적인 분류 문제에서는 정답만 1, 나머지는 0인 One-hot 레이블(Hard Label)을 쓴다. 예를 들어 이미지가 "고양이"라면:

```
Hard Label: [0, 0, 1, 0, 0]   # 고양이 클래스만 1
```

반면 Teacher 모델이 출력하는 Soft Label은 이렇게 생겼다:

```
Soft Label: [0.02, 0.01, 0.85, 0.10, 0.02]  # 고양이 85%, 강아지 10%, ...
```

Soft Label에는 **클래스 간의 유사도 정보**가 담겨 있다. "고양이와 강아지는 닮았다"는 관계를 Teacher가 이미 내면화하고 있고, Student는 이 분포를 모방함으로써 같은 관계를 더 빠르게 학습한다.

---

## Temperature Scaling

Hinton의 원래 논문에서는 Softmax에 **Temperature(T)** 파라미터를 도입해서 분포를 부드럽게 만든다.

$$
q_i = \frac{\exp(z_i / T)}{\sum_j \exp(z_j / T)}
$$

- T = 1: 일반 Softmax
- T > 1: 분포가 납작해짐 → 클래스 간 확률 차이가 줄어들어 Soft Label이 됨
- T < 1: 분포가 뾰족해짐 → Hard Label에 가까워짐

학습 시 Teacher와 Student 모두 같은 T를 적용해서 Soft Label을 구한 뒤, KL Divergence(쿨백-라이블러 발산)를 최소화하는 방향으로 Student를 학습시킨다.

---

## 손실 함수

Knowledge Distillation의 최종 Loss는 두 항을 결합한다:

```
L = α * L_CE(y, y_hard) + (1 - α) * L_KD(T)
```

| 항 | 의미 |
|---|---|
| `L_CE(y, y_hard)` | 정답 레이블에 대한 일반 Cross Entropy Loss |
| `L_KD(T)` | Teacher Soft Label과 Student 예측 사이의 KL Divergence |
| `α` | 두 Loss를 조절하는 가중치 (하이퍼파라미터) |

실제로는 `L_KD`에 `T²`를 곱해서 스케일을 맞춰주는 경우가 많다 (T가 커지면 그래디언트가 작아지기 때문).

---

## 구현 예시 (PyTorch)

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

def distillation_loss(student_logits, teacher_logits, labels, T=4.0, alpha=0.5):
    # Soft Target Loss (KL Divergence)
    soft_targets = F.softmax(teacher_logits / T, dim=1)
    soft_predictions = F.log_softmax(student_logits / T, dim=1)
    kd_loss = F.kl_div(soft_predictions, soft_targets, reduction='batchmean') * (T ** 2)

    # Hard Target Loss (Cross Entropy)
    ce_loss = F.cross_entropy(student_logits, labels)

    return alpha * ce_loss + (1 - alpha) * kd_loss


# 학습 루프 예시
for inputs, labels in dataloader:
    with torch.no_grad():
        teacher_logits = teacher_model(inputs)  # Teacher는 추론만

    student_logits = student_model(inputs)
    loss = distillation_loss(student_logits, teacher_logits, labels, T=4.0, alpha=0.5)

    optimizer.zero_grad()
    loss.backward()
    optimizer.step()
```

---

## Distillation의 종류

### 1. Response-based Distillation
- Teacher의 최종 출력(Logit 또는 Soft Probability)만 활용
- 가장 단순하고 구현이 쉬움
- Hinton 원래 논문의 방식

### 2. Feature-based Distillation
- Teacher 중간 레이어의 특징 맵(Feature Map)을 Student가 모방
- FitNets (2015)에서 제안
- 더 깊은 표현을 전달할 수 있지만, Teacher-Student 아키텍처가 달라지면 차원을 맞추는 작업이 필요

```python
# Feature-based 예시 (차원이 맞지 않을 때 Adapter Layer 사용)
adapter = nn.Linear(student_feature_dim, teacher_feature_dim)
feature_loss = F.mse_loss(adapter(student_features), teacher_features.detach())
```

### 3. Relation-based Distillation
- 데이터 샘플 간의 **관계(Relation)** 를 Teacher에서 Student로 전달
- 예: 배치 내 두 샘플의 임베딩 거리 분포를 Student가 흉내
- RKD(Relational Knowledge Distillation, 2019) 등이 대표적

---

## Self-Distillation과 Online Distillation

### Self-Distillation
Teacher와 Student가 같은 아키텍처일 때도 Distillation이 효과적임. 깊은 레이어가 얕은 레이어를 가르치는 구조로, 별도 Teacher 없이 스스로 개선하는 방식이다. Born Again Networks(2018)가 대표 사례.

### Online Distillation
Teacher와 Student가 동시에 학습하면서 서로의 지식을 주고받는 방식. 사전에 학습된 Teacher가 필요 없다. DML(Deep Mutual Learning, 2018)이 대표적이다.

---

## LLM에서의 Knowledge Distillation

GPT, LLaMA처럼 수백억 파라미터짜리 LLM을 경량화할 때도 Distillation이 핵심 기법으로 쓰인다.

| 방법 | 설명 |
|---|---|
| DistilBERT | BERT를 6레이어(절반)로 줄이면서 97% 성능 유지 |
| TinyLLaMA | LLaMA 아키텍처를 1.1B 파라미터로 줄인 모델 |
| Alpaca | GPT-4 출력을 학습 데이터로 활용해 LLaMA를 파인튜닝 |

LLM에서는 단순 Logit 기반 Distillation 외에도, Teacher의 **Chain-of-Thought(추론 과정)** 을 Student에게 가르치는 방식(Reasoning Distillation)도 연구되고 있다.

---

## Distillation vs 다른 경량화 기법 비교

| 기법 | 방식 | 특징 |
|---|---|---|
| Knowledge Distillation | 작은 모델이 큰 모델의 출력을 학습 | 정확도 유지율 높음 |
| 양자화(Quantization) | 가중치 비트 수를 줄임 (FP32 → INT8) | 하드웨어 친화적, 별도 재학습 불필요 가능 |
| 프루닝(Pruning) | 중요도 낮은 가중치/뉴런 제거 | 구조적 Sparse 처리 필요 |
| 저랭크 분해(Low-rank Factorization) | 행렬을 저랭크로 근사 | 선형 레이어 압축에 적합 |

실제 배포 환경에서는 이 기법들을 조합해서 사용하는 경우가 많다. 예를 들어 Distillation으로 먼저 소형 모델을 만들고, 이후 양자화를 추가로 적용하는 식이다.

---

## 정리

- Knowledge Distillation은 Teacher의 Soft Label(확률 분포)을 이용해 Student를 학습시키는 경량화 기법
- Temperature를 높여 분포를 부드럽게 만들면 클래스 간 관계 정보가 잘 전달됨
- Response-based / Feature-based / Relation-based 세 가지 방식으로 나뉨
- LLM 시대에도 DistilBERT, TinyLLaMA처럼 핵심 기법으로 계속 활용됨
- 양자화, 프루닝과 조합하면 더 강력한 경량화 효과를 낼 수 있음
