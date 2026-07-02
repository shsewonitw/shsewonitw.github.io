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

## AI Agent란

AI Agent는 단순히 질문에 답하는 것을 넘어, **목표를 달성하기 위해 스스로 계획을 세우고 행동(action)을 수행하는 시스템**이다.

일반 LLM 호출과의 차이:

| 구분 | 일반 LLM | AI Agent |
|------|----------|----------|
| 입력 | 프롬프트 | 목표(goal) |
| 출력 | 텍스트 응답 | 행동 결과 |
| 반복 | 단발성 | 루프(loop) 반복 |
| 도구 사용 | 불가 | 가능 (Tool Use) |

Agent는 **"생각 → 행동 → 관찰 → 다시 생각"** 의 사이클을 반복하면서 목표에 도달한다.

---

## ReAct 패턴

Agent의 대표적인 동작 방식이 **ReAct(Reasoning + Acting)** 패턴이다.

```
Thought: 현재 상황을 분석하고 무엇을 해야 할지 판단
Action: 도구를 호출하거나 행동 수행
Observation: 행동 결과를 확인
Thought: 결과를 바탕으로 다음 단계 판단
...
Answer: 최종 답 도출
```

예시 — "서울 날씨 알려줘"를 처리하는 Agent:

```
Thought: 현재 서울 날씨를 모르니 날씨 API를 호출해야겠다
Action: weather_tool(city="Seoul")
Observation: {"temp": 28, "condition": "맑음"}
Thought: 결과를 받았으니 답변을 생성할 수 있다
Answer: 서울은 현재 28도이며 맑습니다
```

---

## Tool Use (도구 사용)

Tool Use는 **LLM이 외부 함수나 API를 호출할 수 있도록 하는 기능**이다.

모델은 텍스트를 직접 생성하는 대신, "이 도구를 이 인자로 호출하라"는 **구조화된 출력**을 내보낸다. 그러면 코드가 실제로 함수를 실행하고 결과를 다시 모델에 넘긴다.

### 기본 흐름

```
1. 사용자 메시지 + 도구 목록 → LLM
2. LLM이 도구 호출 결정 → tool_call 출력
3. 호스트 코드가 실제 함수 실행
4. 실행 결과 → LLM에 다시 전달
5. LLM이 결과를 바탕으로 최종 응답 생성
```

### 도구 정의 예시 (Anthropic API)

```json
{
  "name": "get_weather",
  "description": "특정 도시의 현재 날씨를 가져온다",
  "input_schema": {
    "type": "object",
    "properties": {
      "city": {
        "type": "string",
        "description": "날씨를 조회할 도시 이름"
      }
    },
    "required": ["city"]
  }
}
```

### 모델이 도구를 선택하는 기준

LLM은 질문의 의도와 도구의 `description`을 매칭해서 호출 여부를 결정한다. 그래서 description을 잘 작성하는 게 중요하다.

---

## Agent의 메모리 구조

Agent가 복잡한 작업을 처리하려면 **상태(state)를 기억**해야 한다.

| 메모리 종류 | 설명 | 예시 |
|------------|------|------|
| In-context memory | LLM의 컨텍스트 윈도우 안에 있는 정보 | 대화 히스토리 |
| External memory | 외부 DB나 벡터 스토어 | RAG 시스템 |
| Episodic memory | 과거 에피소드 기록 | "지난번에 A를 했으니..." |
| Procedural memory | 어떻게 행동할지에 대한 지식 | 시스템 프롬프트 |

단기 작업은 컨텍스트 윈도우만으로 처리 가능하지만, 장기 작업이나 대용량 데이터는 외부 메모리(벡터 DB 등)를 붙여야 한다.

---

## 멀티 에이전트 시스템

복잡한 작업은 하나의 Agent가 모두 처리하는 대신 **여러 Agent를 조합**하는 방식으로 해결한다.

```
Orchestrator Agent
├── Research Agent  (검색 담당)
├── Code Agent      (코드 작성 담당)
└── Review Agent    (검토 담당)
```

### 구성 패턴

- **Sequential**: Agent A의 출력이 Agent B의 입력으로 전달 (파이프라인)
- **Parallel**: 여러 Agent가 동시에 병렬 실행
- **Hierarchical**: Orchestrator가 하위 Agent들을 지휘

멀티 에이전트는 역할 분리로 각 Agent가 더 집중된 작업을 수행할 수 있고, 출력 품질이 올라간다. 반면 Agent 간 통신 비용과 오류 전파 위험이 있다.

---

## 실제 구현 예시 (Python, Anthropic SDK)

```python
import anthropic
import json

client = anthropic.Anthropic()

tools = [
    {
        "name": "get_stock_price",
        "description": "특정 주식의 현재 가격을 가져온다",
        "input_schema": {
            "type": "object",
            "properties": {
                "ticker": {"type": "string", "description": "주식 티커 심볼 (예: AAPL)"}
            },
            "required": ["ticker"]
        }
    }
]

def get_stock_price(ticker: str) -> dict:
    # 실제로는 외부 API 호출
    prices = {"AAPL": 189.5, "GOOGL": 175.2}
    return {"ticker": ticker, "price": prices.get(ticker, 0)}

messages = [{"role": "user", "content": "애플 주식 현재가가 얼마야?"}]

while True:
    response = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=1024,
        tools=tools,
        messages=messages
    )

    if response.stop_reason == "end_turn":
        print(response.content[0].text)
        break

    # 도구 호출 처리
    tool_use = next(b for b in response.content if b.type == "tool_use")
    result = get_stock_price(**tool_use.input)

    messages.append({"role": "assistant", "content": response.content})
    messages.append({
        "role": "user",
        "content": [{
            "type": "tool_result",
            "tool_use_id": tool_use.id,
            "content": json.dumps(result)
        }]
    })
```

---

## Agent 설계 시 고려사항

**루프 종료 조건**을 명확히 정의해야 한다. 무한 루프에 빠지지 않도록 최대 반복 횟수나 타임아웃을 설정한다.

**오류 처리**: 도구 호출이 실패했을 때 Agent가 어떻게 대응할지 결정해야 한다. 재시도, 대안 도구 사용, 에러 메시지 반환 등.

**할루시네이션 방지**: Agent가 도구를 호출하지 않고 가상의 결과를 만들어내는 경우가 있다. 도구 결과 없이는 확정적인 답을 내리지 않도록 프롬프트 설계가 필요하다.

**비용 관리**: Agent 루프가 길어질수록 API 호출 횟수와 토큰이 늘어난다. 단계별 비용 추적이 필요하다.
