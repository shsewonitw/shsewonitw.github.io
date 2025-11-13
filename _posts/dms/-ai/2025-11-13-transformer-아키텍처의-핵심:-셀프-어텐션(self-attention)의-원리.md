---
layout: post
title: "[Daily morning study] Transformer 아키텍처의 핵심: 셀프 어텐션(Self-Attention)의 원리"
description: >
  #daily morning study
category: 
    - dms
    - -ai
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---
# Transformer 아키텍처의 핵심: 셀프 어텐션(Self-Attention)의 원리

Transformer 아키텍처는 자연어 처리에서 혁신을 가져온 모델입니다. 이 문서에서는 Transformer의 핵심 구성 요소인 셀프 어텐션이 어떻게 작동하는지를 살펴보겠습니다.

## 1. Transformer 아키텍처 개요

Transformer는 주로 두 가지 구성 요소로 이루어져 있습니다:

1. **인코더** (Encoder)
2. **디코더** (Decoder)

인코더는 입력 시퀀스를 처리하여 정보를 추출하고, 디코더는 이를 바탕으로 출력 시퀀스를 생성합니다. 이 두 가지 구성 요소는 셀프 어텐션 메커니즘을 사용하여 입력 간의 관계를 모델링합니다.

## 2. 셀프 어텐션(Self-Attention)이란?

셀프 어텐션은 입력 시퀀스 내의 각 단어가 다른 단어와 어떻게 상호작용하는지를 이해하게 해주는 메커니즘입니다. 예를 들어, "그는 집에 갔다. 그의 집은 크다."라는 문장에서 "그의"가 "그"를 가리킬 때, 셀프 어텐션 메커니즘은 이를 인식하여 의미를 잘 파악할 수 있게 합니다.

### 2.1. 쿼리(Query), 키(Key), 값(Value)

셀프 어텐션의 기본 아이디어는 각 단어를 쿼리, 키, 값의 개념으로 변환하는 것입니다.

- **쿼리(Query)**: 현재 단어에서 다른 단어와의 유사성을 평가하기 위해 사용됩니다.
- **키(Key)**: 다른 단어를 표현하는 벡터입니다. 해당 키는 쿼리와 비교하여 선택됩니다.
- **값(Value)**: 해당 키에 연결된 정보를 담고 있는 벡터입니다. 최종 출력에 기여합니다.

### 2.2. 셀프 어텐션 메커니즘

셀프 어텐션 메커니즘은 다음과 같은 절차로 작동합니다:

1. **입력 임베딩**: 각 단어를 임베딩 벡터로 변환합니다.
2. **쿼리, 키, 값 생성**: 입력 임베딩을 통해 쿼리, 키, 값을 생성합니다.
   \[
   Q = XW^Q, \quad K = XW^K, \quad V = XW^V
   \]
   여기서 \(X\)는 입력 임베딩이고, \(W^Q, W^K, W^V\)는 학습 가능한 가중치 행렬입니다.

3. **스케일된 점곱 어텐션**: 쿼리와 키 간의 유사성을 계산합니다.
   \[
   \text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V
   \]
   여기서 \(d_k\)는 키 벡터의 차원 수입니다. 스케일링은 그래디언트 폭주를 방지하는 역할을 합니다.

4. **출력**: 최종적으로 값 벡터(V)의 가중 합이 출력으로 생성됩니다.

## 3. 셀프 어텐션의 장점

- **병렬 처리**: RNN과 달리 입력 데이터의 모든 위치를 동시에 처리할 수 있어 학습 속도가 빠릅니다.
- **장기 의존성**: 먼 거리의 단어 간의 관계를 쉽게 파악할 수 있습니다.
- **가변 길이 처리**: 입력 시퀀스의 길이에 구애받지 않고 동작합니다.

## 4. 코드 예시

Python의 PyTorch 라이브러리를 사용하여 셀프 어텐션을 구현해보겠습니다.

```python
import torch
import torch.nn.functional as F

def scaled_dot_product_attention(query, key, value):
    """
    스케일된 점곱 어텐션 계산
    """
    d_k = query.size(-1)  # 쿼리의 차원 수
    scores = torch.matmul(query, key.transpose(-2, -1)) / torch.sqrt(d_k)
    weights = F.softmax(scores, dim=-1)
    output = torch.matmul(weights, value)
    return output

# 예제 입력
query = torch.rand(1, 5, 64)  # (배치 크기, 입력 길이, 임베딩 차원)
key = torch.rand(1, 5, 64)
value = torch.rand(1, 5, 64)

output = scaled_dot_product_attention(query, key, value)
print(output.shape)  # (1, 5, 64) 출력 형상
```

## 5. 결론

셀프 어텐션 메커니즘은 Transformer의 강력한 특징으로, 자연어 처리에서 뛰어난 성능을 발휘하는 이유 중 하나입니다. 이 메커니즘은 입력 시퀀스의 내적 관계를 효율적으로 인식하여 모델이 더 똑똑하게 학습하도록 돕습니다.

## 참고 자료

- "Attention is All You Need" (Vaswani et al., 2017)
- PyTorch Documentation: [torch.nn.MultiheadAttention](https://pytorch.org/docs/stable/generated/torch.nn.MultiheadAttention.html) 

이 문서를 통해 셀프 어텐션의 원리를 잘 이해하고 활용할 수 있기를 바랍니다.
