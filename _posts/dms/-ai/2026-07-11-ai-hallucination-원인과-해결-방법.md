---
layout: post
title: "[Daily morning study] AI Hallucination 원인과 해결 방법"
description: >
  #daily morning study
category: 
    - dms
    - dms-ai
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## AI Hallucination이란

LLM(대형 언어 모델)이 사실과 다른 정보를 매우 그럴듯하게 출력하는 현상을 **Hallucination(환각)**이라고 한다.  
모델이 거짓말을 하는 게 아니라, 학습된 패턴을 기반으로 "있을 법한" 답을 생성하다 보니 틀린 내용이 자신 있게 나온다.

대표적인 예시:
- 존재하지 않는 논문 인용
- 없는 API 함수명 제시
- 잘못된 날짜/수치/인물 정보
- 실제로 없는 법률 조항 인용

---

## 왜 Hallucination이 발생하는가

### 1. 확률적 텍스트 생성 구조

LLM은 이전 토큰들을 보고 다음 토큰의 확률 분포를 계산해서 토큰을 하나씩 뽑는다.  
"정답을 기억해서 출력"하는 게 아니라 "다음에 올 법한 단어를 예측"하는 방식이라 사실과 어긋날 수 있다.

```
P(token_n | token_1, token_2, ..., token_{n-1})
```

### 2. 학습 데이터의 한계

- 학습 데이터에 오류나 편향이 포함됨
- 최신 정보는 학습 데이터에 없음 (knowledge cutoff)
- 희귀한 주제일수록 훈련 샘플이 적어 부정확해짐

### 3. 모델이 "모른다"고 말하도록 훈련받지 않음

RLHF 과정에서 인간이 자신 있는 답변을 선호하는 경향이 있다.  
그 결과 모델이 불확실한 경우에도 확신을 가진 것처럼 답변하는 방향으로 편향된다.

### 4. Context Window 문제

긴 문서나 대화에서 앞부분의 정보를 제대로 반영하지 못하고 잘못된 내용을 생성하기도 한다.

---

## Hallucination의 유형

| 유형 | 설명 | 예시 |
|------|------|------|
| Factual Hallucination | 사실과 다른 정보 생성 | "세종대왕은 1397년 태어났다" (실제 1397년 맞지만 그외 날짜 오류) |
| Fabrication | 존재하지 않는 것을 만들어냄 | 없는 논문, 없는 API |
| Intrinsic Hallucination | 주어진 컨텍스트와 모순 | 문서 내용과 정반대 요약 |
| Extrinsic Hallucination | 컨텍스트 외부 정보 추가 | 입력에 없는 정보를 요약에 포함 |

---

## 해결 방법

### 1. RAG (Retrieval-Augmented Generation)

외부 지식베이스에서 관련 문서를 검색해서 컨텍스트로 제공하는 방식.  
모델이 기억에 의존하지 않고 실제 문서를 참조하므로 사실 오류가 줄어든다.

```
[사용자 질문] → [검색 엔진] → [관련 문서 검색] → [문서 + 질문을 LLM에 전달] → [답변]
```

한계: 검색된 문서 자체가 부정확하거나, 검색이 실패하면 여전히 환각 발생 가능.

### 2. Grounding (근거 제시 강제)

프롬프트 설계로 "근거 없이 답하지 말 것", "모르면 모른다고 해" 등의 지시를 추가한다.

```
당신은 주어진 문서만을 참고해서 답해야 합니다. 
문서에 없는 내용은 "해당 정보를 찾을 수 없습니다"라고 답하세요.
```

효과적이지만 완벽하지는 않다. 모델이 지시를 어기기도 한다.

### 3. Self-Consistency

같은 질문을 여러 번 샘플링해서 가장 많이 나온 답변을 선택한다.  
일관되지 않은 hallucination은 걸러질 가능성이 높다.

```python
responses = [model.generate(prompt) for _ in range(5)]
# majority voting 또는 semantic clustering
final_answer = majority_vote(responses)
```

### 4. Chain-of-Thought (CoT) 프롬프팅

"단계별로 생각해 보세요" 지시를 추가해서 중간 추론 과정을 출력하게 만든다.  
추론 단계가 명시되면 중간에 오류를 잡을 수 있고, 전반적인 사실 정확도가 올라간다.

### 5. Fine-tuning & RLHF 개선

- 정확한 데이터로 파인튜닝해서 특정 도메인의 환각을 줄임
- "모른다"는 응답에도 긍정적 피드백을 주는 방식으로 RLHF 재설계

### 6. Output 검증 파이프라인

LLM 출력을 그대로 사용하지 않고 별도 검증 레이어를 추가한다.

```
[LLM 출력] → [사실 검증 모델 or 검색 검증] → [검증 통과 시 반환 / 실패 시 재생성]
```

---

## 실무에서의 대응 전략

**도메인이 중요한 서비스**라면 RAG + Grounding을 기본으로 깔아야 한다.  
예: 의료, 법률, 금융 도메인은 hallucination 하나로 큰 피해가 생김.

**일반 챗봇**이라면 온도(temperature)를 낮추고 CoT를 활용하는 것만으로도 효과적이다.  
temperature가 낮을수록 확률이 높은(=안전한) 토큰을 선택하는 경향이 있다.

**모델 선택**도 영향을 미친다. 최신 모델일수록 자체적인 불확실성 인식(calibration)이 개선돼 있다.

---

## 정리

Hallucination은 LLM의 구조적 한계에서 비롯된 문제라 완전히 제거하기는 어렵다.  
대신 RAG, Grounding, 검증 파이프라인 같은 외부 장치로 실용적 수준까지 낮추는 게 현실적인 접근이다.  
중요한 의사결정에 LLM을 사용할 때는 항상 human-in-the-loop 구조를 갖추는 게 좋다.
