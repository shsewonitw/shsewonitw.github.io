---
layout: post
title: "[AI] 브라우저에서 실행하는 LLM"
subtitle: "WebLLM"
category: 
    - devlog
    - ai
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/cc9860ca-b4e5-4161-a50f-698508eac228)

---

# WebLLM — 브라우저에서 LLM을 실행하는 기술과 활용 방법

대규모 언어 모델(LLM)은 이제 브라우저에서도 실행할 수 있습니다.  
**WebLLM**은 WebGPU와 WebAssembly를 활용해 서버 없이도 LLM을 실행합니다.

## WebLLM이란?

- 브라우저에서 완전 오프라인으로 LLM 실행 가능
- 개인정보 보호 및 빠른 응답 가능

## 어떻게 동작하는가?

- 모델은 처음 로드 후 CacheStorage에 저장
- WebAssembly + WebGPU로 빠른 실행

## 장점 비교

| 항목 | 클라우드 LLM | WebLLM |
| ---- | ----------- | ------ |
| 오프라인 사용 | 불가능 | 가능 |
| 개인정보 보호 | 서버 전송 필요 | 로컬 실행 |
| 설치 | 서버 필요 | 브라우저만 있으면 가능 |

## 사용 예

```bash
npm i @mlc-ai/web-llm
```

```javascript
const engine = await webllm.CreateMLCEngine("모델명");
const reply = await engine.chat.completions.create({ messages });
```

## 결론

WebLLM은 가벼운 AI 서비스나 개인화된 애플리케이션 개발에 매우 적합합니다.
