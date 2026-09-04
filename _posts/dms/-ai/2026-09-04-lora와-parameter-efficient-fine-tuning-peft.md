---
layout: post
title: "[Daily morning study] LoRA와 PEFT (Parameter-Efficient Fine-Tuning)"
description: >
  #daily morning study
category: 
    - dms
    - dms-ai
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 왜 PEFT가 필요한가

LLM을 특정 도메인에 맞게 파인튜닝하려면 전체 파라미터를 업데이트해야 한다. GPT-3는 175B, LLaMA-2는 70B 파라미터를 갖는다. 이 모델들을 Full Fine-Tuning하면 엄청난 GPU 메모리와 시간이 소모된다.

PEFT(Parameter-Efficient Fine-Tuning)는 모델 전체 파라미터 중 **극히 일부만 학습**해서 Full Fine-Tuning에 준하는 성능을 달성하는 방법들의 총칭이다.

---

## LoRA (Low-Rank Adaptation)

### 핵심 아이디어

2021년 Microsoft가 발표한 논문에서 제안한 방법. 핵심 가정은 다음과 같다.

> 사전학습된 모델의 가중치 행렬은 **낮은 내재적 차원(low intrinsic dimension)**을 가진다. 즉, 파인튜닝 과정에서 실제로 변화하는 정보량은 전체 파라미터 수에 비해 훨씬 작다.

이 가정 아래, 가중치 변화량 ΔW를 두 개의 저랭크 행렬로 분해한다.

```
W' = W + ΔW = W + B × A
```

- `W`: 원래 사전학습 가중치 (고정, 학습 안 함)
- `A`: shape `(r, d_in)` — 랜덤 가우시안으로 초기화
- `B`: shape `(d_out, r)` — 0으로 초기화
- `r`: rank (하이퍼파라미터, 보통 4~64)

초기화할 때 B=0이므로 학습 시작 시 ΔW=0이 보장된다. 즉, 학습 초기에는 원래 모델과 동일하게 동작한다.

### 파라미터 절감 효과

원래 가중치 행렬이 `d × k` 크기라면:

| 방식 | 학습 파라미터 수 |
|------|----------------|
| Full Fine-Tuning | d × k |
| LoRA (rank r) | r × (d + k) |

예: d=4096, k=4096, r=8 이면

- Full: 16,777,216개
- LoRA: 65,536개 → **약 256배 감소**

### 추론 시 오버헤드 없음

학습이 끝나면 `W + B×A`를 미리 합쳐서 W에 병합할 수 있다. 따라서 추론 단계에서는 추가 연산이 전혀 없다.

```python
# 학습 후 가중치 병합 예시
merged_weight = pretrained_W + lora_B @ lora_A
```

---

## 주요 PEFT 기법 비교

### 1. Adapter

Transformer의 각 레이어에 작은 병목(bottleneck) 레이어를 삽입한다. 오직 이 어댑터 레이어만 학습한다.

```
[기존 레이어] → [Down-projection] → [Non-linear] → [Up-projection] → [기존 레이어 출력에 더함]
```

- 장점: 구현이 직관적
- 단점: 추론 시 어댑터 연산이 직렬로 추가되어 레이턴시 증가

### 2. Prefix Tuning

입력 시퀀스 앞에 학습 가능한 "가상 토큰(prefix)"을 붙여 모델 동작을 유도한다. 프롬프트 튜닝과 유사하지만 모든 레이어에 prefix를 추가한다.

```
[PREFIX 토큰들] + [실제 입력 토큰들] → Transformer
```

- 장점: 모델 구조 변경 없음
- 단점: 시퀀스 길이가 늘어나 컨텍스트 창 소모

### 3. Prompt Tuning

Prefix Tuning의 단순화 버전. 입력 임베딩 레이어에만 학습 가능한 soft prompt를 추가한다.

- 장점: 가장 파라미터 수가 적음
- 단점: 모델이 클수록 성능이 좋고, 작은 모델에서는 Full Fine-Tuning 대비 성능 차이가 큼

### 4. IA³ (Infused Adapter by Inhibiting and Amplifying Inner Activations)

어텐션의 키(K), 값(V), 피드포워드 레이어의 활성화에 학습 가능한 스케일링 벡터를 곱하는 방식. LoRA보다 파라미터 수가 훨씬 적다.

---

## 기법 비교 요약

| 기법 | 추가 파라미터 | 추론 오버헤드 | 구현 복잡도 |
|------|-------------|-------------|-----------|
| Full Fine-Tuning | 전체 | 없음 | 낮음 |
| Adapter | 적음 | 있음 (직렬) | 보통 |
| Prefix Tuning | 적음 | 있음 (컨텍스트) | 보통 |
| Prompt Tuning | 매우 적음 | 있음 (컨텍스트) | 낮음 |
| LoRA | 적음 | 없음 (병합 가능) | 낮음 |
| IA³ | 매우 적음 | 없음 | 낮음 |

---

## QLoRA

QLoRA는 LoRA에 4비트 양자화(Quantization)를 결합한 방법이다. 2023년 발표되었으며, **단일 48GB GPU에서 65B 파라미터 모델**을 파인튜닝할 수 있게 해줬다.

핵심 구성 요소:
- **NF4 (NormalFloat4)**: 정규분포를 가정한 4비트 데이터 타입
- **Double Quantization**: 양자화 상수를 다시 양자화해 메모리를 더 절약
- **Paged Optimizers**: GPU OOM 발생 시 CPU 메모리로 자동 오프로드

```python
# QLoRA 사용 예시 (Hugging Face + PEFT 라이브러리)
from transformers import AutoModelForCausalLM, BitsAndBytesConfig
from peft import LoraConfig, get_peft_model

bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_use_double_quant=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.bfloat16
)

model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-2-7b-hf",
    quantization_config=bnb_config,
    device_map="auto"
)

lora_config = LoraConfig(
    r=8,               # rank
    lora_alpha=32,     # 스케일링 계수
    target_modules=["q_proj", "v_proj"],  # 적용할 레이어
    lora_dropout=0.1,
    bias="none"
)

model = get_peft_model(model, lora_config)
model.print_trainable_parameters()
# 출력 예: trainable params: 4,194,304 || all params: 6,742,609,920 || trainable%: 0.0622
```

---

## LoRA의 주요 하이퍼파라미터

| 파라미터 | 의미 | 보통 값 |
|---------|------|--------|
| `r` (rank) | 저랭크 행렬의 차원. 클수록 표현력↑, 메모리↑ | 4, 8, 16, 64 |
| `lora_alpha` | 스케일링 계수. 실제 스케일은 `alpha/r` | 16, 32 |
| `target_modules` | LoRA를 적용할 레이어 이름 | `q_proj`, `v_proj` 등 |
| `lora_dropout` | 과적합 방지 드롭아웃 비율 | 0.05~0.1 |

`lora_alpha/r` 비율이 실질적인 학습률 스케일처럼 작동한다. alpha=r일 때 스케일 1.0이다.

---

## 어떤 레이어에 LoRA를 적용할까

Transformer의 Attention 가중치에 주로 적용한다:

- `q_proj` (Query 행렬)
- `v_proj` (Value 행렬)
- `k_proj` (Key 행렬)
- `o_proj` (Output 행렬)
- 피드포워드 레이어: `gate_proj`, `up_proj`, `down_proj`

논문에서는 Q와 V에만 적용해도 충분한 성능을 얻을 수 있다고 보고한다. 모든 레이어에 적용하면 성능은 더 좋지만 그만큼 학습 파라미터가 늘어난다.

---

## LoRA 변형들

- **LoRA+**: A와 B의 학습률을 다르게 설정해 더 효율적인 학습
- **DoRA (Weight-Decomposition LoRA)**: 가중치를 크기(magnitude)와 방향(direction)으로 분해해 LoRA 적용
- **AdaLoRA**: 레이어별로 rank를 중요도에 따라 자동으로 조절
- **LoftQ**: 양자화와 LoRA 초기화를 함께 최적화

---

## 실무 선택 기준

- **메모리가 넉넉하고 성능이 중요** → Full Fine-Tuning
- **GPU 메모리가 제한적이고 추론 속도가 중요** → LoRA
- **극단적으로 적은 메모리, 대형 모델** → QLoRA
- **태스크 간 전환이 잦음(멀티태스크)** → Adapter 또는 LoRA (각 태스크마다 어댑터 교체)
- **프롬프트만으로 간단히 적용** → Prompt Tuning

LoRA는 현재 가장 널리 쓰이는 PEFT 기법이다. HuggingFace의 `peft` 라이브러리에서 몇 줄로 적용할 수 있고, 학습 후 가중치를 병합하면 추론 오버헤드도 없다. 실제 프로덕션에서 도메인 특화 모델을 만들 때 QLoRA + LoRA 조합이 사실상 표준으로 자리잡았다.
