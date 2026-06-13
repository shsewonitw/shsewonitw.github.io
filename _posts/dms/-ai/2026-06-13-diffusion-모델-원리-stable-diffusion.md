---
layout: post
title: "[Daily morning study] Diffusion 모델 원리 (Stable Diffusion)"
description: >
  #daily morning study
category: 
    - dms
    - dms-ai
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## Diffusion 모델이란

Diffusion 모델은 이미지를 점진적으로 노이즈로 만든 다음, 역방향으로 노이즈를 제거하는 과정을 학습해서 새로운 이미지를 생성하는 생성 모델이다. GAN이 Generator와 Discriminator의 경쟁으로 학습하는 것과 달리, Diffusion은 노이즈 예측이라는 단순한 목표 하나로 학습하기 때문에 학습이 안정적이고 생성 품질도 높다.

---

## Forward Process (노이즈 추가)

원본 이미지 x₀에 T 스텝에 걸쳐 가우시안 노이즈를 조금씩 추가해서, 최종적으로 완전한 노이즈 xT로 만드는 과정이다.

각 스텝의 수식:

```
q(x_t | x_{t-1}) = N(x_t; √(1-β_t) * x_{t-1}, β_t * I)
```

- `β_t`: 각 스텝에서 추가할 노이즈의 강도 (noise schedule)
- β_t가 작으면 조금씩, 크면 많이 노이즈를 추가
- 일반적으로 T=1000 스텝 사용

x₀에서 임의의 스텝 t로 바로 점프하는 것도 가능하다:

```
q(x_t | x_0) = N(x_t; √ᾱ_t * x_0, (1-ᾱ_t) * I)
```

여기서 `ᾱ_t = ∏(1-β_s)` (s=1부터 t까지의 누적곱). 덕분에 학습 시 임의의 t 스텝 샘플을 한 번에 만들 수 있다.

---

## Reverse Process (노이즈 제거)

완전한 노이즈 xT에서 출발해서 단계적으로 노이즈를 제거하며 이미지를 복원하는 과정이다. 각 스텝에서 신경망(U-Net)이 현재 노이즈 이미지 x_t와 시간 스텝 t를 받아 추가된 노이즈 ε를 예측한다.

```
p_θ(x_{t-1} | x_t) = N(x_{t-1}; μ_θ(x_t, t), Σ_θ(x_t, t))
```

---

## 학습 목표

실제로 추가된 노이즈 ε와 신경망이 예측한 노이즈 ε_θ 사이의 MSE를 최소화한다.

```
L = E[||ε - ε_θ(x_t, t)||²]
```

학습 흐름:
1. 원본 이미지 x₀ 샘플링
2. 랜덤 타임스텝 t, 가우시안 노이즈 ε 샘플링
3. x_t = √ᾱ_t * x₀ + √(1-ᾱ_t) * ε 계산
4. U-Net에 x_t, t를 입력해 ε_θ 예측
5. L = ||ε - ε_θ||² 로 파라미터 업데이트

---

## Stable Diffusion 아키텍처

기본 Diffusion 모델은 pixel space에서 동작하기 때문에 고해상도 이미지에서 계산 비용이 매우 크다. Stable Diffusion은 이를 latent space에서 수행하도록 개선한 **Latent Diffusion Model (LDM)** 이다.

### 4가지 핵심 컴포넌트

**1. VAE (Variational Autoencoder)**

이미지를 압축된 latent space로 변환하고 복원하는 역할이다.

- Encoder: 이미지(512×512×3) → latent(64×64×4), 8배 압축
- Decoder: latent(64×64×4) → 이미지(512×512×3)
- Diffusion은 이 latent space에서만 동작하므로 계산량이 대폭 줄어든다

**2. U-Net**

latent space에서 각 스텝의 노이즈를 예측하는 메인 신경망이다.

- Encoder + Bottleneck + Decoder 구조, Skip Connection으로 연결
- ResBlock: 각 블록에서 잔차 연결로 학습 안정화
- Attention Layer: Self-Attention과 Cross-Attention 포함
- Time Embedding: 현재 스텝 t를 사인파로 인코딩해 네트워크에 전달

**3. CLIP Text Encoder**

텍스트 프롬프트를 벡터로 변환하는 모듈이다.

- OpenAI CLIP 모델의 텍스트 인코더 활용
- 토큰화 → transformer 인코딩 → text embedding
- 이 embedding이 U-Net의 Cross-Attention에 condition으로 들어간다

**4. Noise Scheduler**

각 스텝의 노이즈 추가/제거 비율을 관리한다.

- DDPM: 원래 1000 스텝, Markovian 확률 샘플링
- DDIM: 20~50 스텝으로 줄인 결정론적 샘플링 (같은 seed → 같은 이미지)
- DPM-Solver: 더 빠른 고차 ODE 기반 솔버

---

## 이미지 생성 흐름

```
텍스트 프롬프트
      ↓
CLIP Text Encoder
      ↓
text embedding ─────────────────────────────────────┐
                                                     ↓
랜덤 가우시안 노이즈 (latent 64×64×4) → U-Net (Cross-Attention) → denoised latent
                                              ↑ (N번 반복)
                                         Noise Scheduler
                                                     ↓
                                            VAE Decoder
                                                     ↓
                                          최종 이미지 (512×512)
```

---

## CFG (Classifier-Free Guidance)

프롬프트를 얼마나 강하게 반영할지 조절하는 핵심 기법이다.

**학습 단계**: 동일한 U-Net이 조건부(텍스트 있음)와 비조건부(텍스트 없음, null 토큰) 예측을 모두 학습한다.

**추론 단계**: 두 예측을 조합해서 최종 노이즈를 계산한다.

```
noise_pred = uncond_pred + guidance_scale × (cond_pred - uncond_pred)
```

- guidance_scale = 1: 비조건부 예측만 사용 (프롬프트 무시)
- guidance_scale = 7.5: 일반적으로 많이 쓰는 값
- guidance_scale이 높을수록 프롬프트를 더 강하게 따르지만 다양성은 감소

---

## DDIM Sampling

DDPM의 느린 샘플링 속도를 해결한 방법이다.

| | DDPM | DDIM |
|---|---|---|
| 샘플링 방식 | Markovian (확률적) | non-Markovian (결정론적) |
| 필요 스텝 | 1000 | 20~50 |
| 재현성 | 같은 seed여도 다를 수 있음 | 같은 seed → 항상 같은 이미지 |
| 속도 | 느림 | 빠름 |

DDIM은 U-Net 가중치를 그대로 쓰면서 샘플링 공식만 바꾼 것이기 때문에, 추가 학습 없이 속도를 크게 개선할 수 있다.

---

## ControlNet

텍스트 프롬프트 외에 추가 이미지 조건(pose, edge, depth 등)으로 생성을 더 세밀하게 제어하는 기법이다.

```
입력: [텍스트 프롬프트] + [제어 이미지 (예: 포즈 스켈레톤)]
      ↓
ControlNet (U-Net 복사본, 학습 가능) + 원본 U-Net (동결)
      ↓
제어 신호를 U-Net 디코더에 주입
      ↓
제어된 이미지 생성
```

원본 U-Net 가중치는 고정(freeze)하고, 그 복사본을 제어 조건으로 학습시킨다. 덕분에 기존 모델 품질을 유지하면서 포즈·구도·선화를 정확히 따르는 이미지를 만들 수 있다.

---

## GAN vs Diffusion 비교

| 특징 | GAN | Diffusion |
|---|---|---|
| 학습 안정성 | mode collapse 문제 있음 | 안정적 |
| 생성 품질 | 높음 | 더 높음 |
| 샘플링 속도 | 빠름 (단일 forward pass) | 느림 (iterative) |
| 출력 다양성 | 제한적 | 높음 |
| 텍스트 조건 제어 | 어려움 | CFG로 쉽게 제어 |

---

## 정리

Diffusion 모델의 핵심은 **"노이즈를 추가하는 과정을 뒤집는 것"** 이다. Forward process는 수식으로 정의된 고정 과정이고, Reverse process만 신경망이 학습한다. Stable Diffusion은 여기에 VAE로 latent 압축, CLIP으로 텍스트 조건, CFG로 가이던스를 추가해서 실용적인 text-to-image 생성 시스템을 구축한 것이다.

학습 때는 단순히 노이즈 예측 MSE를 최소화하는데, 추론 때는 그 U-Net을 수십 번 반복 호출해서 점진적으로 이미지를 만들어 낸다는 점이 GAN과 가장 큰 구조적 차이다.
