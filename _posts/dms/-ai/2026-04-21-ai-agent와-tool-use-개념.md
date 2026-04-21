---
layout: post
title: "[Daily morning study] AI Agent와 Tool Use 개념"
description: >
  #daily morning study
category: 
    - dms
    - dms-ai
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## AI Agent란?

LLM을 단순히 질문-응답 도구로 쓰는 게 아니라, 목표를 달성하기 위해 스스로 계획을 세우고 도구를 사용하며 반복적으로 행동하는 시스템을 Agent라고 한다.

전통적인 LLM 사용은 입력 → 출력 한 번으로 끝난다. Agent는 여기서 더 나아가 다음 루프를 돈다.

1. 목표를 받는다
2. 어떤 도구(Tool)를 써야 할지 판단한다
3. 도구를 실행하고 결과를 받는다
4. 결과를 바탕으로 다음 행동을 결정한다
5. 목표가 달성될 때까지 반복한다

---

## Agent의 구성 요소

| 구성 요소 | 역할 |
|----------|------|
| LLM (Brain) | 추론, 계획, 의사결정 |
| Memory | 이전 대화/행동 이력 저장 |
| Tools | 외부 API 호출, 검색, 코드 실행 등 |
| Planning | 목표를 하위 작업으로 분해 |

Memory는 크게 두 종류다.

- **단기 메모리**: 현재 대화 컨텍스트 (context window 내)
- **장기 메모리**: 외부 데이터베이스나 벡터 스토어에 영구 저장

---

## Tool Use (Function Calling)

Tool Use는 LLM이 자신이 직접 답을 만드는 대신, 외부 함수를 호출해서 결과를 가져오는 메커니즘이다. OpenAI는 "Function Calling", Anthropic Claude는 "Tool Use"라고 부른다.

### 작동 순서

1. 사용자가 질문을 보낸다
2. LLM이 "이 질문은 외부 도구가 필요하다"고 판단한다
3. LLM이 어떤 함수를, 어떤 인자로 호출할지 JSON 형태로 응답한다
4. 애플리케이션이 실제 함수를 실행한다
5. 실행 결과를 다시 LLM에게 넘긴다
6. LLM이 최종 응답을 생성한다

### Anthropic Tool Use 예시

```python
import anthropic

client = anthropic.Anthropic()

tools = [
    {
        "name": "get_weather",
        "description": "특정 도시의 현재 날씨를 반환한다.",
        "input_schema": {
            "type": "object",
            "properties": {
                "city": {
                    "type": "string",
                    "description": "날씨를 조회할 도시명"
                }
            },
            "required": ["city"]
        }
    }
]

response = client.messages.create(
    model="claude-opus-4-7",
    max_tokens=1024,
    tools=tools,
    messages=[{"role": "user", "content": "서울 날씨 알려줘"}]
)

if response.stop_reason == "tool_use":
    tool_use = next(b for b in response.content if b.type == "tool_use")
    tool_name = tool_use.name    # "get_weather"
    tool_input = tool_use.input  # {"city": "서울"}

    weather_result = call_weather_api(tool_input["city"])

    final_response = client.messages.create(
        model="claude-opus-4-7",
        max_tokens=1024,
        tools=tools,
        messages=[
            {"role": "user", "content": "서울 날씨 알려줘"},
            {"role": "assistant", "content": response.content},
            {
                "role": "user",
                "content": [
                    {
                        "type": "tool_result",
                        "tool_use_id": tool_use.id,
                        "content": weather_result
                    }
                ]
            }
        ]
    )
```

---

## ReAct 패턴

ReAct = **Re**asoning + **Act**ing. Agent가 어떻게 생각하고 행동하는지를 명시적으로 구조화한 패턴이다.

```
Thought: 사용자가 서울 날씨를 묻고 있다. get_weather 도구를 써야 한다.
Action: get_weather(city="서울")
Observation: {"temperature": 18, "condition": "맑음"}
Thought: 결과를 받았다. 사용자에게 전달하면 된다.
Answer: 서울의 현재 날씨는 맑음이고, 기온은 18도입니다.
```

Thought → Action → Observation 사이클을 반복하면서 최종 답에 도달한다. LLM이 중간 추론 과정을 텍스트로 생성하게 만들어 복잡한 문제를 단계적으로 풀 수 있게 한다.

---

## 자주 쓰이는 Tool 유형

- **웹 검색**: 최신 정보 조회
- **코드 실행 (Code Interpreter)**: Python 코드 실행 후 결과 반환
- **파일 시스템**: 파일 읽기/쓰기
- **외부 API 호출**: 날씨, 주식, DB 쿼리 등
- **벡터 스토어 검색**: RAG 파이프라인에서 관련 문서 검색

---

## Multi-Agent 시스템

단일 Agent 대신 여러 Agent가 협력하는 구조다.

```
Orchestrator Agent
    ├── Research Agent   (웹 검색 전담)
    ├── Coding Agent     (코드 작성 전담)
    └── Review Agent     (코드 리뷰 전담)
```

각 Agent가 특정 역할에 집중하므로 복잡한 작업을 분업 처리할 수 있다. 단점은 Agent 간 통신 비용, 오류 전파 가능성, 디버깅 어려움이다.

---

## Agent의 한계

- **Hallucination**: Tool을 써야 할 상황에서 꾸며낸 답을 그냥 반환할 수 있다
- **무한 루프**: 목표를 달성 못하면 도구 호출을 끝없이 반복한다
- **비용**: Tool 호출마다 LLM API가 한 번 더 호출되므로 토큰 비용이 누적된다
- **보안**: 외부 API를 호출하거나 파일을 수정할 때 예상치 못한 사이드 이펙트가 생길 수 있다

이를 방지하기 위해 최대 반복 횟수 제한(`max_iterations`), Tool 접근 권한 제한, Human-in-the-loop(중간 결과를 사람이 검토하는 단계) 같은 안전장치를 건다.
