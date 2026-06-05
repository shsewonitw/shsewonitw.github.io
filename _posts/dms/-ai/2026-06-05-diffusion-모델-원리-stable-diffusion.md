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

Diffusion 모델은 이미지를 점진적으로 노이즈로 망가뜨린 뒤, 그 역과정을 학습시켜 노이즈에서 원본 이미지를 복원하는 생성 모델이다. Stable Diffusion, DALL·E 2, Imagen 등이 모두 이 계열이다.

핵심 아이디어는 두 단계로 나뉜다.

1. **Forward process (확산)** — 원본 이미지에 가우시안 노이즈를 T 스텝에 걸쳐 조금씩 더해, 결국 순수한 가우시안 노이즈 상태로 만든다.
2. **Reverse process (복원)** — 노이즈 상태에서 시작해 스텝마다 노이즈를 예측·제거하며 원본에 가까운 이미지를 복원한다.

---

## Forward Process

시간 t에서 이미지 x_t는 이전 이미지 x_{t-1}에 약간의 노이즈를 추가한 것이다.

```
x_t = √(1 - βt) * x_{t-1} + √βt * ε,   ε ~ N(0, I)
```

βt는 스텝마다의 노이즈 강도(스케줄)이고, T 스텝이 지나면 x_T는 거의 순수한 가우시안 분포가 된다.

수식을 전개하면 임의의 t 스텝에서의 x_t를 원본 x_0으로부터 닫힌 형태(closed form)로 바로 계산할 수 있다.

```
x_t = √ᾱt * x_0 + √(1 - ᾱt) * ε
```

ᾱt는 β1~βt의 누적곱으로 정의된다. 학습 중에 임의의 t를 샘플링해서 x_t를 한 번에 만들 수 있어 효율적이다.

---

## Reverse Process (U-Net의 역할)

모델은 x_t와 시간 t를 입력받아 해당 스텝에서 추가된 노이즈 ε를 예측하도록 학습된다. 실제 구현에서는 **U-Net** 아키텍처가 주로 쓰인다.

학습 손실은 단순히 실제 노이즈 ε와 예측 노이즈 ε_θ 사이의 MSE다.

```
L = E[||ε - ε_θ(x_t, t)||²]
```

추론 시에는 순수 가우시안 노이즈 x_T에서 시작해 T → 0 방향으로 스텝마다 노이즈를 제거하며 최종 이미지를 만든다.

---

## Stable Diffusion의 핵심: Latent Diffusion

픽셀 공간에서 직접 Diffusion을 수행하면 고해상도 이미지일수록 연산 비용이 폭발적으로 증가한다. Stable Diffusion은 이를 **잠재 공간(latent space)** 에서 수행해 해결한다.

전체 파이프라인은 세 컴포넌트로 구성된다.

| 컴포넌트 | 역할 |
|---------|------|
| VAE (Variational Autoencoder) | 이미지를 저차원 잠재 벡터로 인코딩 / 복원 |
| U-Net (+ Attention) | 잠재 공간에서 Diffusion의 노이즈 예측 |
| Text Encoder (CLIP 등) | 텍스트 프롬프트를 임베딩으로 변환해 U-Net에 조건 제공 |

1. 이미지를 VAE 인코더로 압축해 latent z를 얻는다 (예: 512×512 → 64×64×4).
2. z에 Forward diffusion을 적용해 노이즈를 추가한다.
3. U-Net이 latent 공간에서 노이즈를 예측하고 제거한다.
4. 복원된 latent를 VAE 디코더로 다시 픽셀 공간으로 변환한다.

텍스트 프롬프트는 Cross-Attention을 통해 U-Net의 각 레이어에 조건으로 주입된다.

---

## Classifier-Free Guidance (CFG)

텍스트 조건을 얼마나 강하게 반영할지 조절하는 기법이다. 같은 U-Net을 두 번 실행한다.

1. 텍스트 조건 있음 → 조건부 노이즈 예측 ε_c
2. 텍스트 조건 없음(빈 프롬프트) → 무조건 노이즈 예측 ε_u

최종 노이즈 예측은 두 결과를 선형 보간한다.

```
ε_final = ε_u + guidance_scale * (ε_c - ε_u)
```

`guidance_scale`(CFG 스케일)이 높을수록 텍스트에 더 충실한 이미지가 생성되지만, 너무 높으면 과포화·왜곡이 발생한다. 보통 7~12 사이를 사용한다.

---

## 샘플링 스케줄러

Reverse process를 T 스텝 전부 수행하면 느리다. 실용적으로는 스케줄러를 통해 스텝 수를 줄인다.

| 스케줄러 | 특징 |
|---------|------|
| DDPM | 원래 논문 방식, 1000 스텝 필요 |
| DDIM | 결정론적 샘플링, 20~50 스텝으로 가능 |
| DPM++ | 고차 솔버, 15~25 스텝에서 좋은 품질 |
| Euler / Euler A | 빠르고 안정적, 실제로 많이 사용 |

DDIM은 동일한 노이즈 시드에서 일관된 이미지를 생성하기 때문에 img2img나 inpainting에도 자주 쓰인다.

---

## img2img와 Inpainting

**img2img**: 입력 이미지를 일부 노이즈화(denoise_strength로 조절)한 뒤 Reverse process를 실행한다. strength가 낮을수록 원본에 가까운 이미지가 나온다.

**Inpainting**: 마스크 영역만 노이즈화하고 나머지는 원본을 유지한 채 Reverse process를 실행한다. 특정 부분만 수정할 때 사용한다.

---

## GAN과 비교

| 항목 | GAN | Diffusion |
|------|-----|-----------|
| 학습 안정성 | 불안정 (mode collapse) | 안정적 |
| 생성 품질 | 날카롭지만 다양성 부족 | 높은 다양성 |
| 속도 | 빠름 (단일 forward pass) | 느림 (다단계 reverse) |
| 조건부 생성 | 추가 구조 필요 | CFG로 자연스럽게 통합 |

Diffusion 모델이 이미지 생성 품질과 텍스트 조건 제어 면에서 GAN을 대부분의 태스크에서 앞지르면서 주류가 됐다.

---

## 정리

- Diffusion 모델은 Forward(노이즈 추가) + Reverse(노이즈 제거)를 학습하는 생성 모델이다.
- Stable Diffusion은 VAE로 픽셀을 latent로 압축한 뒤 latent 공간에서 Diffusion을 수행해 효율을 높인다.
- CLIP 기반 텍스트 인코더와 Cross-Attention으로 텍스트 조건부 생성을 구현한다.
- CFG 스케일로 텍스트 충실도를 조절하고, DDIM/DPM++ 같은 스케줄러로 샘플링 속도를 높인다.
