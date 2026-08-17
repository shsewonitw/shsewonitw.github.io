---
layout: post
title: "[Daily morning study] Mixture of Experts (MoE) 아키텍처 — 대형 언어 모델의 효율적 확장"
description: >
  #daily morning study
category: 
    - dms
    - dms-ai
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## Mixture of Experts (MoE)란?

**MoE(Mixture of Experts)**는 모델 전체를 항상 다 쓰는 게 아니라, 입력마다 일부 "전문가(Expert)" 서브넷만 선택적으로 활성화하는 아키텍처다.

기존 Dense 모델(GPT-3, LLaMA 등)은 추론할 때 모든 파라미터가 계산에 참여한다. 반면 MoE는 수십 ~ 수백 개의 Expert를 두고, 각 토큰마다 그 중 소수만 활성화한다. 덕분에 **파라미터 총량은 크지만 실제 연산량은 훨씬 적다.**

대표적인 MoE 기반 모델:

- **GPT-4** (OpenAI, 아키텍처 공개 안 됨 — MoE 추정)
- **Mixtral 8x7B** (Mistral AI, 오픈소스)
- **Gemini 1.5** (Google)
- **Switch Transformer** (Google Brain, 2021)

---

## 핵심 개념: Expert와 Router

### Expert

Expert는 일반적으로 Transformer의 **Feed-Forward Network(FFN) 레이어를 여러 개 복제**한 것이다. 각 Expert는 독립적인 가중치를 갖는다.

```
Layer 구조 (Dense):
  Attention → FFN → 다음 레이어

Layer 구조 (MoE):
  Attention → Router → [Expert_1, Expert_2, ..., Expert_N] 중 일부 선택 → 다음 레이어
```

### Router (Gating Network)

Router는 각 토큰에 대해 "어떤 Expert를 써야 하나?"를 결정하는 작은 선형 레이어다.

```python
# 개념적인 Router 동작
def router(token_hidden_state, num_experts, top_k):
    # 각 Expert에 대한 점수 계산
    scores = linear(token_hidden_state)  # shape: [num_experts]
    
    # 상위 k개 Expert 선택
    top_k_scores, top_k_indices = topk(softmax(scores), k=top_k)
    
    # 선택된 Expert의 출력을 가중합
    return top_k_indices, top_k_scores
```

top-k 값은 보통 1 또는 2를 사용한다. Mixtral 8x7B는 8개 Expert 중 2개를 선택(top-2).

---

## Mixtral 8x7B 구조 살펴보기

Mixtral 8x7B는 이름과 달리 단순히 7B짜리 8개가 아니다.

실제 구조:

- 총 파라미터: **46.7B**
- FFN Expert 수: **8개**
- 토큰당 활성화 Expert: **2개**
- 활성 파라미터(추론 시): **~12.9B** (Attention + 2 Expert)

즉, 46.7B 파라미터를 가지지만 추론 시에는 약 13B 수준의 연산만 일어난다. 이 덕분에 LLaMA 2 70B와 비슷한 성능을 훨씬 적은 연산으로 낸다.

---

## MoE의 장점

| 항목 | Dense 모델 | MoE 모델 |
|------|-----------|---------|
| 총 파라미터 | 상대적으로 적음 | 많음 |
| 추론 연산량 | 전체 파라미터 사용 | 일부 Expert만 사용 |
| 학습 속도 | 느림 | 빠름 (같은 연산량 대비) |
| 메모리 | 낮음 | 높음 (전체 Expert 로딩 필요) |

핵심 장점은 **"같은 연산 비용으로 더 큰 모델을 학습 가능"** 하다는 것이다. Expert를 늘릴수록 모델의 표현력은 커지지만, 추론당 활성화되는 연산량은 일정하게 유지된다.

---

## 주요 문제점: Load Balancing

MoE의 가장 큰 문제는 **특정 Expert에 트래픽이 몰리는 현상**이다.

Router가 항상 몇몇 Expert만 선호하면, 나머지 Expert는 학습이 잘 안 된다(Dead Expert 문제). 이를 막기 위해 다양한 기법이 사용된다.

### 보조 손실(Auxiliary Loss)

Expert 간 부하가 균등해지도록 손실에 페널티를 추가한다.

```python
# Switch Transformer의 auxiliary loss 개념
def load_balance_loss(router_probs, expert_indices, num_experts):
    # 각 Expert가 실제로 선택된 비율 f_i
    expert_fraction = fraction_of_tokens_per_expert(expert_indices, num_experts)
    
    # 각 Expert에 대한 Router 평균 확률 p_i
    expert_prob_avg = mean_router_prob_per_expert(router_probs, num_experts)
    
    # 두 값이 고르게 분포할수록 loss가 낮아짐
    loss = num_experts * sum(expert_fraction * expert_prob_avg)
    return loss
```

### Expert Capacity

각 Expert가 한 배치에서 처리할 수 있는 최대 토큰 수를 제한한다. 용량을 초과하면 해당 토큰은 다음 Expert로 넘어가거나 그냥 통과(Pass-through)된다.

---

## 분산 추론에서의 MoE

MoE는 Expert를 서로 다른 GPU에 분산할 수 있어 **Expert Parallelism**이 가능하다.

```
GPU 0: Expert 1, 2
GPU 1: Expert 3, 4
GPU 2: Expert 5, 6
GPU 3: Expert 7, 8
```

각 토큰은 Router를 통해 어떤 GPU의 Expert로 갈지 결정된다. 단, 이때 GPU 간 통신(All-to-All)이 필요해 통신 비용이 생긴다.

반면 Attention 레이어는 모든 GPU가 동일하게 가지고 있어야 하므로 (Tensor Parallelism 또는 복제), 메모리 요구량이 높다. 이것이 MoE의 단점 중 하나다.

---

## Dense vs MoE 비교 요약

```
Dense Transformer FFN:
  x → Linear(d_model → d_ff) → GELU → Linear(d_ff → d_model)
  
  모든 토큰이 동일한 FFN을 통과

MoE Transformer FFN:
  x → Router → top-k Expert 선택
     → Expert_i: Linear → GELU → Linear (i ∈ top-k)
     → 선택된 Expert 출력의 가중합

  토큰마다 다른 Expert가 활성화됨
```

---

## 왜 MoE가 주목받는가

GPT-4가 MoE 기반이라는 소문이 퍼지면서 "어떻게 모델 크기를 키우면서도 추론 비용을 낮추는가?"가 업계의 핵심 질문이 됐다.

MoE는 그 답 중 하나다. 같은 추론 FLOPs에서 Dense 모델보다 훨씬 많은 파라미터(= 더 풍부한 지식)를 담을 수 있기 때문이다. Mixtral 8x7B가 오픈소스로 공개되면서 MoE가 실용적으로 쓸 수 있는 기술임이 증명됐다.

앞으로 LLM은 점점 더 MoE 방식으로 수렴할 가능성이 높다. 단, 메모리 사용량(모든 Expert를 VRAM에 올려야 함)이 여전히 걸림돌이라 엣지 디바이스보다는 대규모 서버 환경에 적합하다.

---

## 핵심 요약

- MoE = 여러 Expert FFN + Router (어떤 Expert 쓸지 결정)
- 토큰마다 top-k Expert만 활성화 → 연산량은 작고 총 파라미터는 큼
- Load Balancing 문제를 보조 손실 + capacity 제한으로 해결
- Mixtral 8x7B: 46.7B 파라미터, 추론 시 ~13B 연산
- Expert Parallelism으로 GPU 여러 장에 분산 가능 → 단, 통신 비용 발생
- Dense 대비 같은 비용으로 더 강력한 모델 학습 가능
