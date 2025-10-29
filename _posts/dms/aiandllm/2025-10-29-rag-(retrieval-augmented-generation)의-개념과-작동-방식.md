---
layout: post
title: "[Daily morning study] RAG (Retrieval-Augmented Generation)의 개념과 작동 방식"
description: >
  #daily morning study
category: 
    - dms
    - "aiandllm"
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---
# RAG (Retrieval-Augmented Generation) 개념과 작동 방식

## RAG란 무엇인가?

RAG는 "Retrieval-Augmented Generation"의 약자로, 정보 검색과 자연어 생성(NLG)을 결합한 기술입니다. 기본적으로, RAG는 대량의 데이터를 효율적으로 활용하여 더 정확하고 정보가 풍부한 텍스트 생성을 목표로 합니다. 이 기술은 특히 검색 시스템과 생성 모델을 통합하여 사용자가 제공한 질문에 대해 더 정확하고 심층적인 답변을 제공할 수 있습니다.

## RAG의 구성 요소

RAG는 크게 두 가지 주요 구성 요소로 이루어져 있습니다:

1. **검색기 (Retriever)**: 사용자 질문에 관련된 문서를 검색하는 모듈로, 대량의 데이터베이스에서 관련 정보를 찾는 역할을 수행합니다.
   
2. **생성기 (Generator)**: 검색된 문서를 기반으로 질문에 대한 답변을 생성하는 모델입니다. 일반적으로 Transformer 기반의 언어 모델이 사용됩니다.

## 작동 방식

RAG의 작동 방식은 다음과 같이 진행됩니다:

1. **질문 입력**: 사용자가 질문을 입력합니다.

2. **문서 검색**:
    - 입력된 질문을 기반으로 검색기를 사용해 관련 문서를 찾습니다.
    - 이 단계에서는 BM25와 같은 검색 알고리즘을 사용할 수 있습니다. BM25는 TF-IDF의 확장으로, 특정 질의와 문서 간의 관련성을 평가 합니다.
  
3. **정보 처리**:
    - 검색된 문서를 생성기로 입력합니다. 이 때 여러 문서를 함께 고려하는 방식으로 적절한 정보를 선택합니다.

4. **답변 생성**:
    - 생성 모델은 입력된 질문과 관련된 문서 정보를 조합하여 자연어로 답변을 생성합니다.

5. **결과 출력**:
    - 최종 생성된 답변을 사용자에게 제공합니다.

### 간단한 코드 예제

아래는 Hugging Face의 `transformers` 라이브러리를 사용하여 RAG 모델을 간단히 구현하는 예제입니다.

```python
from transformers import RagTokenizer, RagRetriever, RagSequenceForGeneration

# 토크나이저와 모델 로드하기
tokenizer = RagTokenizer.from_pretrained("facebook/rag-sequence-nq")
model = RagSequenceForGeneration.from_pretrained("facebook/rag-sequence-nq")

# 질문 입력
question = "What is RAG?"

# 질문을 토크나이즈하고 모델에 입력
input_ids = tokenizer(question, return_tensors="pt").input_ids

# 답변 생성
generated_ids = model.generate(input_ids)
answer = tokenizer.batch_decode(generated_ids, skip_special_tokens=True)

print(answer)
```

## RAG의 장점

- **정확성 향상**: 단순히 모델의 내부 지식에 의존하지 않고, 외부 데이터를 활용해 보다 정확한 답변을 생성합니다.
- **정보의 다양성**: 다양한 출처에서 정보를 수집하고 요약할 수 있어 답변의 다양성과 깊이를 높입니다.
- **유연성**: 사용자가 제공하는 질문의 형태나 내용에 따라 적응이 가능하여, 다양한 용도로 활용될 수 있습니다.

## RAG의 한계

- **검색 품질 의존성**: 검색기가 제대로 작동하지 않으면 생성된 답변의 품질도 떨어질 수 있습니다.
- **실시간 데이터 반영 어려움**: 결과가 사전 학습된 데이터에 기반하기 때문에 최신 정보를 반영하는 데 한계가 있습니다.

## 결론

RAG는 검색과 생성 모델을 결합하여 보다 유용한 정보 제공을 목표로 하는 혁신적인 접근 방식입니다. 이 기술은 많은 자연어 처리(NLP) 응용 프로그램에서 활용될 수 있으며, 더욱 정확하고 유익한 대화를 가능하게 합니다. RAG의 원리를 이해하고 활용한다면, 다양한 질문에 대한 효과적인 답변 생성에 큰 도움이 될 것입니다.
