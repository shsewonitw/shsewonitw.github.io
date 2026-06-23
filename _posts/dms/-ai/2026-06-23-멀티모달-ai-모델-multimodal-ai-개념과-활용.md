---
layout: post
title: "[Daily morning study] 멀티모달 AI 모델 개념과 활용"
description: >
  #daily morning study
category: 
    - dms
    - dms-ai
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## 멀티모달 AI란

멀티모달(Multimodal) AI는 텍스트 하나만 입력받는 기존 언어 모델과 달리, **두 가지 이상의 다른 형식(modality)** 데이터를 함께 처리하는 모델을 말한다.

흔히 다루는 모달리티:

- **텍스트** (Text)
- **이미지** (Image)
- **오디오** (Audio)
- **비디오** (Video)
- **문서/PDF** (Document)

예를 들어 "이 사진에서 무엇이 이상한지 설명해줘"라고 이미지와 텍스트를 함께 전달하면 답변을 생성하는 것이 멀티모달 AI의 핵심 능력이다.

---

## 대표 모델

| 모델 | 개발사 | 지원 모달리티 |
|------|--------|--------------|
| GPT-4o | OpenAI | 텍스트, 이미지, 오디오 |
| Gemini 1.5 Pro | Google DeepMind | 텍스트, 이미지, 오디오, 비디오, 문서 |
| Claude 3.5 Sonnet | Anthropic | 텍스트, 이미지, 문서 |
| LLaVA | 오픈소스 | 텍스트, 이미지 |
| Flamingo | DeepMind | 텍스트, 이미지 |

---

## 핵심 아키텍처: Vision-Language Model (VLM)

이미지+텍스트를 함께 처리하는 Vision-Language Model의 일반적인 구조는 다음과 같다.

```
이미지 입력
    ↓
[Vision Encoder]   ← 이미지를 벡터로 변환 (ViT, CLIP 등)
    ↓
[Projection Layer] ← 이미지 임베딩을 언어 모델 공간으로 매핑
    ↓
[LLM Backbone]     ← 텍스트 토큰 + 이미지 임베딩을 함께 처리
    ↓
텍스트 출력
```

### Vision Encoder

이미지를 수치 벡터로 변환하는 역할이다.

**ViT (Vision Transformer)**: 이미지를 고정 크기의 패치(patch)로 나눈 뒤 각 패치를 토큰처럼 취급해서 Transformer로 처리한다.

```
이미지 (224×224)
  → 16×16 패치로 분할 → 196개 패치
  → 각 패치를 1D 벡터로 flatten
  → Transformer Encoder 통과
  → 이미지 임베딩 벡터 출력
```

**CLIP (Contrastive Language-Image Pre-Training)**: OpenAI가 개발한 모델로, 이미지와 텍스트를 같은 임베딩 공간에 매핑한다. "고양이 사진"이라는 텍스트와 고양이 이미지의 벡터가 가깝게 위치하도록 대조 학습(Contrastive Learning)한다.

### Projection Layer

Vision Encoder에서 나온 이미지 벡터는 차원이나 분포가 LLM의 텍스트 임베딩과 다르다. Projection Layer(보통 Linear Layer 또는 MLP)가 이 간격을 맞춰준다.

### LLM Backbone

GPT, LLaMA 등 일반 언어 모델을 그대로 사용하되, 텍스트 토큰 시퀀스에 이미지 임베딩을 함께 concat해서 입력한다.

```
입력 시퀀스: [IMG_TOKEN_1, IMG_TOKEN_2, ..., "이 사진에서", "무엇이", "보이나요?"]
                ← 이미지 임베딩 →     ← 텍스트 토큰 임베딩 →
```

---

## 학습 방식

### 사전학습 (Pre-training)

대량의 이미지-텍스트 쌍 데이터로 이미지 설명 생성 또는 대조 학습을 수행한다. 인터넷에서 수집한 이미지와 해당 alt 텍스트, 캡션 등이 학습 데이터로 쓰인다.

### 파인튜닝 (Fine-tuning)

특정 태스크(VQA, 문서 이해, 이미지 분류 등)에 맞게 추가 학습한다.

### Instruction Tuning

"이 이미지를 설명해줘", "이 그래프에서 최솟값은?"처럼 자연어 명령을 따르도록 학습시킨다. LLaVA가 이 방식으로 GPT-4가 생성한 이미지 관련 QA 데이터를 활용해 학습했다.

---

## 주요 활용 사례

### 이미지 이해 및 VQA (Visual Question Answering)

```
입력: [상품 사진] + "이 제품의 이름과 특징을 요약해줘"
출력: "파란색 무선 헤드폰으로, 노이즈 캔슬링 기능과 30시간 배터리를 갖추고 있습니다..."
```

### 문서/PDF 이해

스캔된 문서, 계약서, 논문 이미지를 입력받아 내용을 분석하거나 질문에 답한다. OCR 없이도 표, 그래프, 손글씨를 처리할 수 있다.

### 코드 생성 (스크린샷 → 코드)

UI 스크린샷을 입력하면 해당 화면을 구현하는 HTML/CSS 코드를 생성한다.

### 의료 이미지 분석

X-ray, MRI 이미지를 분석해 이상 여부를 판단하거나 보고서를 작성하는 데 활용된다.

### 오디오 멀티모달 (GPT-4o)

음성 입력을 텍스트로 변환하지 않고 직접 오디오 토큰으로 처리해 감정, 억양, 배경 소리까지 이해한다.

---

## 성능 평가 벤치마크

| 벤치마크 | 측정 내용 |
|---------|---------|
| VQAv2 | 이미지 기반 질의응답 정확도 |
| MMMU | 대학 수준 멀티모달 이해력 |
| DocVQA | 문서 이미지 이해 |
| TextVQA | 이미지 내 텍스트 인식 및 이해 |
| COCO Captions | 이미지 캡션 생성 품질 |

---

## 한계와 과제

**Hallucination**: 이미지에 없는 내용을 텍스트로 생성하는 현상. 예를 들어 이미지에 없는 사람을 "있다"고 답하거나, 숫자를 잘못 읽는 경우가 빈번하다.

**파인 그레인드 인식 한계**: 이미지 내 작은 텍스트, 복잡한 도표, 세밀한 수식 등은 여전히 오류율이 높다.

**고해상도 처리 비용**: 고해상도 이미지를 처리하려면 패치 수가 늘어나 컨텍스트 길이와 연산량이 크게 증가한다.

**비디오 처리**: 비디오는 프레임 수가 많아 전체를 처리하기보다 키 프레임을 샘플링해서 사용하는데, 이로 인해 빠르게 변하는 장면에서 정보 손실이 생긴다.

---

## 정리

멀티모달 AI는 Vision Encoder로 이미지를 벡터화한 뒤 Projection Layer를 통해 언어 모델 공간으로 매핑하고, 텍스트 토큰과 함께 LLM에 입력하는 구조가 핵심이다. CLIP 기반 Vision Encoder와 Instruction Tuning의 결합으로 이미지 이해, 문서 분석, 코드 생성 등 다양한 실용적 애플리케이션이 가능해졌다. 다만 Hallucination 문제와 고해상도 처리 비용은 여전히 해결 중인 과제다.
