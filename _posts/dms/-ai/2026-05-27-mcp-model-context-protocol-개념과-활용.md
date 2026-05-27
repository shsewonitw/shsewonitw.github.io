---
layout: post
title: "[Daily morning study] MCP (Model Context Protocol) 개념과 활용"
description: >
  #daily morning study
category: 
    - dms
    - dms-ai
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## MCP란 무엇인가

MCP(Model Context Protocol)는 Anthropic이 2024년에 공개한 오픈 프로토콜로, AI 모델이 외부 도구·데이터 소스·서비스에 **표준화된 방식**으로 연결할 수 있게 해준다.

MCP 이전에는 Claude를 파일 시스템에 연결하거나 GitHub API를 붙이려면 매번 커스텀 구현이 필요했다. MCP는 AI 애플리케이션과 외부 시스템 사이의 "공통 언어"를 정의해서 이 문제를 해결한다.

---

## 핵심 구성 요소

MCP는 클라이언트-서버 아키텍처를 따른다.

| 구성요소 | 역할 |
|---------|------|
| MCP Host | Claude Desktop, IDE 등 AI를 사용하는 애플리케이션 |
| MCP Client | Host 내에서 Server와 1:1 연결을 관리하는 컴포넌트 |
| MCP Server | 외부 데이터/도구를 노출하는 경량 서버 |
| Transport Layer | stdio, HTTP+SSE 등 실제 통신 방식 |

```
Host ↔ Client ↔ (Transport) ↔ Server ↔ External Resource
```

---

## MCP Server가 제공하는 세 가지 기능

### 1. Tools (도구)

AI가 직접 호출할 수 있는 함수다. 부작용이 있을 수 있어서 Host 앱이 사용자에게 승인을 요청하는 흐름이 표준화되어 있다.

```json
{
  "name": "read_file",
  "description": "파일 내용을 읽어 반환",
  "inputSchema": {
    "type": "object",
    "properties": {
      "path": { "type": "string" }
    },
    "required": ["path"]
  }
}
```

### 2. Resources (리소스)

파일, DB 레코드, API 응답 등 AI가 읽을 수 있는 데이터다. URI로 식별한다.

```
file:///home/user/project/README.md
postgres://localhost/mydb/users/123
```

### 3. Prompts (프롬프트)

재사용 가능한 프롬프트 템플릿이다. 복잡한 작업 흐름을 표준화하고 공유할 수 있다.

---

## 통신 방식 (Transport)

MCP는 **JSON-RPC 2.0**을 기반으로 메시지를 교환한다.

**stdio Transport**
- 로컬 프로세스 간 통신 (표준 입출력 사용)
- Claude Desktop 같은 로컬 앱에 적합

```
Host → [stdin]  → Server Process
Host ← [stdout] ← Server Process
```

**HTTP + SSE Transport**
- 원격 서버와 통신
- Client → Server: HTTP POST
- Server → Client: Server-Sent Events
- 웹 기반 서비스에 적합

---

## MCP Server 구현 예시 (Python)

```python
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp import types

server = Server("my-server")

@server.list_tools()
async def list_tools() -> list[types.Tool]:
    return [
        types.Tool(
            name="get_weather",
            description="특정 도시의 날씨를 조회",
            inputSchema={
                "type": "object",
                "properties": {
                    "city": {"type": "string", "description": "도시 이름"}
                },
                "required": ["city"]
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[types.TextContent]:
    if name == "get_weather":
        city = arguments["city"]
        # 실제로는 외부 날씨 API를 호출
        return [types.TextContent(type="text", text=f"{city}의 날씨: 맑음, 22°C")]
    raise ValueError(f"Unknown tool: {name}")

async def main():
    async with stdio_server() as (read_stream, write_stream):
        await server.run(
            read_stream,
            write_stream,
            server.create_initialization_options()
        )

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

---

## Claude Desktop에서 MCP Server 설정

`claude_desktop_config.json`에 서버를 등록한다.

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/Users/user/projects"
      ]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_xxx"
      }
    }
  }
}
```

---

## 대표 공식 MCP Server

| 서버 패키지 | 기능 |
|-------------|------|
| `server-filesystem` | 로컬 파일 시스템 읽기/쓰기 |
| `server-github` | GitHub 저장소, PR, 이슈 관리 |
| `server-postgres` | PostgreSQL 데이터베이스 조회 |
| `server-brave-search` | 웹 검색 |
| `server-slack` | Slack 메시지 전송/조회 |

모두 `@modelcontextprotocol/` 네임스페이스 아래에 있다.

---

## Tool Use vs MCP

Tool Use(Function Calling)는 AI 모델 API 레벨에서 도구를 호출하는 기능이다. MCP는 그 위 레이어에서 도구를 **어떻게 정의하고 제공할지** 표준화한 프로토콜이다.

```
[AI 모델] ──── Tool Use (API 레벨)
                      ↕
               [MCP Protocol]
                      ↕
      [External Tools / Data Sources]
```

Tool Use가 "무엇을 호출할 수 있는가"라면, MCP는 "그 도구를 어떻게 표준화해서 제공할 것인가"다. MCP를 쓰면 한 번 만든 서버를 Claude뿐 아니라 MCP 호환 클라이언트 어디서든 재사용할 수 있다.

---

## MCP의 장점 정리

- **표준화**: 한 번 만든 MCP Server를 여러 AI 앱에서 재사용 가능
- **보안**: 서버가 노출하는 리소스·도구 범위를 명확히 제한, 승인 흐름 표준화
- **생태계**: Slack, Notion, Linear, Jira, Stripe 등 수백 개의 커뮤니티 서버가 이미 존재

---

## 정리

- MCP = AI와 외부 세계를 연결하는 표준 프로토콜 (Anthropic, 2024)
- Server는 Tools / Resources / Prompts 세 가지를 노출
- JSON-RPC 2.0 기반, stdio 또는 HTTP+SSE로 통신
- Claude Desktop·Claude Code 등이 MCP Host로 동작
- Tool Use가 "호출 메커니즘"이라면, MCP는 "도구 제공 표준"
