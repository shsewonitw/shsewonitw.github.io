---
layout: post
title: "[Daily morning study] Vector Database란 무엇인가? (Embedding과의 관계)"
description: >
  #daily morning study
category: 
    - dms
    - -ai
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---
# Vector Database란 무엇인가? (Embedding과의 관계)

## 1. Vector Database의 정의

Vector Database는 대량의 데이터에서 유용한 정보를 빠르게 검색하고 관리하기 위해 데이터를 벡터 형식으로 저장하고 처리하는 데이터베이스입니다. 일반적으로 검색은 고차원 벡터 공간에서 이루어지며, 이 벡터들은 보통 자연어 처리(NLP), 이미지 인식 등의 분야에서 생성된 임베딩(embedding)과 관련이 있습니다.

이러한 데이터베이스는 비슷한 특징을 가진 데이터를 그룹화하여, 사용자가 입력한 쿼리와 가장 유사한 데이터를 빠르게 검색할 수 있도록 설계됩니다.

## 2. Embedding이란?

Embedding은 고차원 데이터를 낮은 차원으로 변환하여 실수 벡터로 표현하는 방법입니다. 이 과정을 통해 데이터 간의 유사성을 효율적으로 측정할 수 있습니다. 예를 들어, 단어 임베딩(word embedding)에서는 단어를 벡터로 변환하여 문맥에 따라 단어 간의 관계를 파악합니다.

일반적으로 Embedding은 딥러닝 모델에서 학습 과정 중에 생성되며, 유사한 의미를 가진 단어들은 가까운 거리의 벡터로 임베딩됩니다.

### 예시: Word2Vec

Word2Vec 모델은 단어를 임베딩하는 대표적인 알고리즘 중 하나입니다. 이 모델은 'Skip-gram' 또는 'CBOW'라는 방식을 사용하여 단어 간의 관계를 학습합니다.

```python
from gensim.models import Word2Vec

# 예시 데이터
sentences = [["I", "love", "machine", "learning"],
             ["I", "love", "to", "learn"],
             ["Natural", "language", "processing", "is", "fun"]]

# Word2Vec 모델 학습
model = Word2Vec(sentences, vector_size=10, window=3, min_count=1, workers=4)

# 'love' 단어의 임베딩
vector = model.wv['love']
print(vector)
```

## 3. Vector Database의 작동 원리

1. **데이터 변환**: 데이터가 입력될 때, 해당 데이터의 간단한 벡터 표현이 생성됩니다. 예를 들어 텍스트 데이터는 NLP 모델을 통해 임베딩을 만들 수 있습니다.

2. **저장**: 생성된 벡터는 벡터 데이터베이스에 저장됩니다. 이 데이터베이스는 고속 검색을 지원하는 구조로 최적화됩니다.

3. **유사도 검색**: 사용자가 쿼리(주로 벡터 형식으로)하면, 데이터베이스는 내장된 유사도 검색 알고리즘을 통해 가장 가까운 벡터를 찾고 관련 데이터를 반환합니다. 일반적으로 코사인 유사도(cosine similarity) 또는 유클리드 거리(Euclidean distance) 기반의 방법이 사용됩니다.

### 예시: 유사도 검색

```python
# 쿼리 문장의 임베딩
query_vector = model.wv['machine']  # 예시로 'machine' 단어의 벡터 사용

# 유사도가 가장 높은 단어 찾기
similar_words = model.wv.most_similar(query_vector, topn=5)
print(similar_words)
```

## 4. Vector Database의 활용 분야

- **추천 시스템**: 사용자 자료와 아이템 정보를 임베딩하여 비슷한 사용자와 아이템을 추천할 수 있음
- **검색 엔진**: 입력 쿼리와 관련된 문서나 제품을 빠르게 검색하여 결과로 반환
- **이미지 검색**: 이미지에서 추출한 특징 벡터를 사용하여 비슷한 이미지를 찾음
- **대화형 AI**: 자연어 처리에서 유저의 질문에 대한 유사한 답변을 찾아주는 역할

## 5. 결론

Vector Database는 고차원 데이터의 효율적인 검색 및 관리를 위한 필수적인 기술로, Embedding과의 밀접한 관계 덕분에 다양한 분야에서 그 가치가 증가하고 있습니다. 데이터를 벡터화하고, 이 벡터를 기반으로 한 검색 시스템은 앞으로도 많은 발전 가능성이 있는 영역입니다. 

이 가이드를 통해 Vector Database와 Embedding의 관계를 명확히 이해하고, 이론뿐 아니라 실제 코드를 통해 구체적인 구현 방법을 익히는데 도움이 되었기를 바랍니다.
