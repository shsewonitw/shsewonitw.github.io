---
layout: post
title: "[Daily morning study] LLM 양자화(Quantization) 기법과 모델 경량화"
description: >
  #daily morning study
category: 
    - dms
    - dms-ai
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 양자화(Quantization)란

대형 언어 모델(LLM)의 파라미터는 기본적으로 32비트 부동소수점(FP32) 또는 16비트 부동소수점(FP16/BF16)으로 저장된다. GPT-3(175B)를 FP32로 저장하면 약 700GB가 필요하다.

**양자화**란 모델 가중치와 활성값을 더 낮은 비트 수로 표현하는 기법이다. 예를 들어 FP32 → INT8로 변환하면 메모리 사용량을 4분의 1로 줄일 수 있다.

```
FP32  : 32bit → 4byte per weight
FP16  : 16bit → 2byte per weight
INT8  : 8bit  → 1byte per weight
INT4  : 4bit  → 0.5byte per weight
```

## 양자화의 기본 원리

연속적인 부동소수점 값을 이산적인 정수 값으로 매핑하는 과정이다.

```
x_quantized = round(x / scale) + zero_point

scale     = (x_max - x_min) / (2^bits - 1)
zero_point = round(-x_min / scale)
```

- **scale**: 원본 값의 범위를 정수 범위로 축소하는 비율
- **zero_point**: 정수 표현에서 0에 해당하는 오프셋
- **de-quantization**: `x = (x_quantized - zero_point) * scale`

## 양자화 방식 분류

### 1. PTQ (Post-Training Quantization)

학습이 끝난 모델을 사후에 양자화한다. 별도의 재학습이 필요 없어 빠르게 적용할 수 있다.

| 방식 | 설명 |
| --- | --- |
| RTN (Round-to-Nearest) | 가중치를 단순 반올림으로 양자화. 가장 빠르지만 정확도 손실이 큼 |
| GPTQ | Hessian 행렬을 활용해 오차를 최소화하며 레이어별로 양자화. INT4에서도 성능 유지 |
| AWQ (Activation-Aware Weight Quantization) | 중요한 가중치 채널을 식별해 보호. 활성값 분포를 고려한 방식 |
| SmoothQuant | 활성값의 이상치를 가중치로 이전시켜 양자화 난이도를 낮춤 |

### 2. QAT (Quantization-Aware Training)

양자화를 시뮬레이션하면서 파인튜닝을 진행한다. PTQ보다 정확도가 높지만 학습 비용이 든다.

```python
# QAT 개념적 흐름
# 순전파: fake quantization 적용 (양자화 오차 반영)
# 역전파: 연속 기울기로 업데이트 (Straight-Through Estimator)
x_fake_quant = dequantize(quantize(x))
loss = criterion(model(x_fake_quant), labels)
loss.backward()
```

## GPTQ 상세

GPTQ(Generative Pre-trained Transformers Quantization)는 LLM PTQ의 대표적인 방법이다.

핵심 아이디어는 **OBQ(Optimal Brain Quantization)**에서 출발한다.

1. 레이어의 가중치를 열 단위로 순서대로 양자화
2. 이미 양자화된 열의 오차를 남은 열에 분산시켜 보정
3. Hessian의 역행렬을 이용해 오차 전파 방향을 계산

```
# 오차 보정 수식 (개념)
W_q[i] = quantize(W[i])
delta = W[i] - W_q[i]
W[j:] -= delta * H_inv[i, j:] / H_inv[i, i]   (j > i)
```

이 덕분에 INT4(4비트)로 압축해도 FP16 대비 성능 저하가 1~2% 수준에 그친다.

## AWQ 상세

AWQ는 "모든 가중치가 동등하게 중요하지 않다"는 관찰에서 출발한다.

- 소수의 **salient(중요) 채널**이 활성화 값에서 이상치(outlier)를 유발한다
- 이 채널들을 보호하거나 스케일링해서 양자화 오차를 줄인다

```
# 중요 채널 스케일링 (단순화)
s = importance_scale(W, X)   # 활성값 분포 기반 스케일 계산
W_scaled = W / s             # 가중치 스케일 다운
X_scaled = X * s             # 활성값 스케일 업 (net effect 동일)
W_q = quantize(W_scaled)     # 스케일된 가중치 양자화
```

GPTQ보다 빠르게 적용 가능하고 속도/정확도 균형이 좋다.

## 비트폭별 특성 요약

| 비트폭 | 메모리 절감 | 정확도 손실 | 주요 활용 |
| --- | --- | --- | --- |
| FP16/BF16 | 2× (vs FP32) | 거의 없음 | 학습, 추론 기본 |
| INT8 | 4× | 매우 적음 | 서버 추론 |
| INT4 | 8× | 적음 (GPTQ/AWQ 사용 시) | 엣지, 모바일 배포 |
| INT2~3 | 10~16× | 눈에 띄는 손실 | 연구 단계 |

## KV Cache 양자화

어텐션의 KV 캐시도 양자화할 수 있다. 긴 시퀀스에서 KV 캐시가 메모리의 상당 부분을 차지하기 때문에 중요하다.

- **KVQuant**, **KIVI** 등의 방법이 제안됨
- 키(Key)와 값(Value)을 채널별로 다른 스케일로 양자화
- FP16 대비 메모리 절반 이하로 줄이면서도 성능 거의 유지

## 혼합 정밀도(Mixed Precision) 양자화

모든 레이어를 같은 비트폭으로 처리하면 성능 저하가 생기는 레이어가 있다. 민감한 레이어(예: 첫 번째/마지막 레이어, 어텐션 레이어)는 고정밀로 유지하고 나머지만 낮은 비트폭을 적용한다.

```
예) 70B 모델
- 어텐션 레이어: INT8
- FFN 레이어: INT4
- 임베딩 레이어: FP16
```

## 실제 사용 예시 (llama.cpp / Transformers)

```python
# Hugging Face bitsandbytes로 INT8 로드
from transformers import AutoModelForCausalLM, BitsAndBytesConfig

quant_config = BitsAndBytesConfig(load_in_8bit=True)
model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-2-7b-hf",
    quantization_config=quant_config,
    device_map="auto"
)
```

```python
# INT4 (NF4) 예시
quant_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",       # NormalFloat4
    bnb_4bit_compute_dtype=torch.bfloat16
)
```

**NF4(NormalFloat4)**는 정규 분포를 따르는 가중치에 최적화된 4비트 데이터 타입으로, QLoRA 논문에서 제안됐다.

## 양자화 vs 다른 경량화 기법 비교

| 기법 | 방식 | 속도 향상 | 정확도 유지 | 재학습 필요 |
| --- | --- | --- | --- | --- |
| 양자화 | 비트 폭 축소 | 높음 | 중간 (기법 의존) | 불필요(PTQ) |
| 프루닝 | 가중치 제거 | 중간 | 중간 | 필요(구조적) |
| 지식 증류 | 소형 모델 학습 | 높음 | 높음 | 필요 |
| LoRA | 저랭크 어댑터 | - | 높음 | 필요(PEFT) |

## 정리

- 양자화는 모델의 **비트 폭을 줄여 메모리와 연산을 절약**하는 기법이다
- PTQ는 재학습 없이 빠르게 적용 가능하며, GPTQ/AWQ가 현재 LLM 배포에 가장 많이 쓰인다
- INT8은 서버 환경에서, INT4는 엣지/온디바이스 배포에서 활용된다
- 비트폭이 낮을수록 메모리 절감이 크지만 정교한 알고리즘 없이는 품질 저하가 생긴다
- KV 캐시 양자화와 혼합 정밀도 방식을 결합하면 더 효과적인 경량화가 가능하다
