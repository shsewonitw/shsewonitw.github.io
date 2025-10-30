---
layout: post
title: "[Daily morning study] LLM 파인튜닝(Fine-tuning)과 프롬프트 엔지니어링(Prompt Engineering)의 차이점"
description: >
  #daily morning study
category: 
    - dms
    - -ai
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---
# LLM 파인튜닝(Fine-tuning)과 프롬프트 엔지니어링(Prompt Engineering)의 차이점

인공지능 분야에서 LLM(대형 언어 모델)의 활용이 증가하면서, 파인튜닝과 프롬프트 엔지니어링 두 가지 기술이 주목받고 있습니다. 이 두 가지 방법은 모델의 성능을 개선하고 특정 작업에 맞게 최적화하는 데 사용되지만, 각기 다른 접근 방식과 특성을 갖고 있습니다. 이 학습 자료에서는 두 기술의 정의, 특성, 장단점, 그리고 실제 예시를 살펴보겠습니다.

## 1. LLM 파인튜닝(Fine-tuning)

### 1.1 정의
파인튜닝은 기본적인 대형 언어 모델을 특정 도메인 또는 작업에 맞게 조정하는 과정입니다. 이미 훈련된 모델에 추가적인 데이터셋을 사용하여 다시 훈련함으로써 모델이 특정한 문맥이나 분야에 대한 이해도를 높이는 방법입니다.

### 1.2 방법
- **기본 모델 선택**: 파인튜닝할 기본 모델을 선택합니다. 예를 들어, GPT-3, BERT 등.
- **데이터 준비**: 해당 작업에 맞는 라벨이 있는 데이터셋을 준비합니다. 예를 들면, 질문 응답 데이터셋.
- **훈련**: 선택한 모델을 데이터셋을 사용해서 일정 기간 재훈련합니다. 이때 이전에 학습한 지식은 활용됩니다.
  
```python
from transformers import GPT2LMHeadModel, GPT2Tokenizer, Trainer, TrainingArguments

# 모델과 토크나이저 로드
model = GPT2LMHeadModel.from_pretrained('gpt2')
tokenizer = GPT2Tokenizer.from_pretrained('gpt2')

# 데이터셋 및 훈련 준비
train_dataset = ...  # 사용자 데이터셋 로드

training_args = TrainingArguments(
    output_dir='./results',
    num_train_epochs=3,
    per_device_train_batch_size=16,
)

trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=train_dataset,
)

# 훈련 시작
trainer.train()
```

### 1.3 장단점
- **장점**: 
  - 특정 도메인에 대한 뛰어난 성능을 발휘할 수 있습니다.
  - 모델이 과거의 지식을 잃지 않고 새로운 정보를 학습합니다.
  
- **단점**:
  - 데이터와 시간 소모가 큽니다.
  - 모델 크기가 커지면 훈련 비용이 증가합니다. 

---

## 2. 프롬프트 엔지니어링(Prompt Engineering)

### 2.1 정의
프롬프트 엔지니어링은 모델의 입력에 효과적으로 질문이나 명령어를 작성하여 원하는 출력을 유도하는 기술입니다. 기존의 언어 모델을 그대로 사용하면서도 다양한 형식의 프롬프트를 통해 성능을 최적화합니다.

### 2.2 방법
- **프롬프트 설계**: 적절한 질문이나 명령어를 디자인합니다. 예를 들어, "이 영화의 주제를 요약해줘."
- **변형 시도**: 프롬프트를 여러 가지 형태로 다듬어 보면서 결과를 비교합니다.

```python
from transformers import GPT2LMHeadModel, GPT2Tokenizer

# 모델과 토크나이저 로드
model = GPT2LMHeadModel.from_pretrained('gpt2')
tokenizer = GPT2Tokenizer.from_pretrained('gpt2')

# 프롬프트 설정
prompt = "다음 문장의 주제를 요약해줘: '인공지능의 발전은 우리의 삶을 크게 변화시키고 있습니다.'"

# 입력 텍스트 토큰화 및 생성
input_ids = tokenizer.encode(prompt, return_tensors='pt')
output = model.generate(input_ids)

# 결과 출력
print(tokenizer.decode(output[0]))
```

### 2.3 장단점
- **장점**: 
  - 훈련이 필요 없고 빠르게 결과를 얻을 수 있습니다.
  - 아주 적은 수의 데이터로도 실험 가능성이 높습니다.
  
- **단점**:
  - 특정 프롬프트에 의존적으로 결과가 다를 수 있습니다.
  - 작업에 따라 최적의 프롬프트를 찾는 것이 어려울 수 있습니다.

---

## 3. LLM 파인튜닝과 프롬프트 엔지니어링 비교

| 특성                     | 파인튜닝                       | 프롬프트 엔지니어링                |
|------------------------|----------------------------|---------------------------------|
| **훈련 여부**          | 필요                       | 불필요                           |
| **데이터 요구량**      | 다량 요구                  | 적은 양으로 가능                   |
| **모델 수정 여부**      | 수정 필요                  | 수정 없이 사용                     |
| **적용 범위**          | 특정 작업에 적합           | 다양한 작업에 활용 가능            |
| **성능 개선 가능성**    | 높음                       | 상황에 따라 다름                    |
| **시간 비용**          | 큼                         | 적음                             |

## 결론

LLM의 활용에 있어 파인튜닝과 프롬프트 엔지니어링은 각기 다른 목적과 상황에 따라 선택할 수 있는 기법입니다. 파인튜닝은 특정 도메인에 대한 높은 정확도를 요구하는 경우에 적합하지만, 프롬프트 엔지니어링은 빠른 실험과 다양한 응답을 얻고자 할 때 유용합니다. 각 방식의 특징을 잘 이해하고 적절히 활용하는 것이 중요합니다.
