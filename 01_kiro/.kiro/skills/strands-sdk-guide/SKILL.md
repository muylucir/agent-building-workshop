---
name: strands-sdk-guide
description: |
  TRIGGER on keywords: Strands, strands-agents, strands SDK, `from strands import`, or any of these class/function names: MCPClient, HookProvider, BeforeToolCallEvent, FileSessionManager, S3SessionManager, SlidingWindowConversationManager, SummarizingConversationManager, structured_output_model, handoff_to_agent, BedrockModel, stream_async.
  Also use when building Python AI agents with tools and model providers (Bedrock, OpenAI, Anthropic) without specifying another framework.
  Covers: agent creation, @tool decorator, MCP server integration, multi-agent (Swarm/Graph/Workflow), hooks, streaming, conversation management, session persistence, structured output with Pydantic.
  DO NOT trigger for: LangChain, LangGraph, CrewAI, OpenAI Assistants, raw boto3, terraform, SageMaker, or AgentCore deploy.
---

# Strands Agents SDK 개발 가이드

Strands Agents SDK는 AI 에이전트를 빠르게 구축할 수 있는 오픈소스 프레임워크다. Python과 TypeScript를 지원한다.

## Agent Loop (핵심 동작 원리)

모델 호출 → 도구 선택 여부 확인 → 도구 실행 → 결과로 다시 모델 호출 → 반복

```python
from strands import Agent
agent = Agent(system_prompt="You are a helpful assistant.")
response = agent("Hello!")
```

## 어떤 문서를 읽어야 하나?

사용자의 요청에 따라 아래 참조 문서 중 필요한 것만 읽는다. 여러 주제에 걸친 요청이면 관련 문서를 모두 읽는다.

| 사용자 요청 | 읽을 문서 |
|------------|----------|
| 설치, 환경설정, 첫 에이전트, AWS 자격증명 | [quickstart.md](references/quickstart.md) |
| 커스텀 도구 생성, MCP 연동, 도구 스트리밍 | [tools.md](references/tools.md) |
| 모델 변경, Bedrock/OpenAI/Anthropic 설정, 캐싱 | [model-providers.md](references/model-providers.md) |
| 멀티 에이전트, Graph/Swarm/Workflow 패턴 | [multi-agent.md](references/multi-agent.md) |
| Hooks, 스트리밍 처리, 콜백 핸들러 | [hooks-streaming.md](references/hooks-streaming.md) |
| 대화 관리, 세션, Structured Output, Observability | [session-output.md](references/session-output.md) |

## 자주 쓰는 레시피

### 레시피 1: 도구가 있는 기본 에이전트

```python
from strands import Agent, tool

@tool
def search_db(query: str) -> str:
    """Search the database for records.

    Args:
        query: Search query string
    """
    return f"Found results for: {query}"

agent = Agent(
    system_prompt="You are a helpful assistant with database access.",
    tools=[search_db]
)
response = agent("Find users named Alice")
```

### 레시피 2: MCP 서버 연동 에이전트

```python
from mcp import stdio_client, StdioServerParameters
from strands import Agent
from strands.tools.mcp import MCPClient

mcp_client = MCPClient(lambda: stdio_client(
    StdioServerParameters(command="uvx", args=["awslabs.aws-documentation-mcp-server@latest"])
))

agent = Agent(tools=[mcp_client])
response = agent("What is AWS Lambda?")
```

### 레시피 3: 멀티 에이전트 협업 (Swarm)

```python
from strands import Agent
from strands.multiagent import Swarm
from strands.tools.swarm import handoff_to_agent

researcher = Agent(name="researcher", system_prompt="Research topics. Hand off to writer when done.", tools=[handoff_to_agent])
writer = Agent(name="writer", system_prompt="Write content based on research.", tools=[handoff_to_agent])

swarm = Swarm(agents=[researcher, writer], initial_agent="researcher")
result = swarm("Write a blog post about AI agents")
```

### 레시피 4: Structured Output

```python
from pydantic import BaseModel, Field
from strands import Agent

class PersonInfo(BaseModel):
    name: str = Field(description="Name")
    age: int = Field(description="Age")

agent = Agent()
result = agent("John is a 30-year-old engineer", structured_output_model=PersonInfo)
print(result.structured_output.name)  # John
```

### 레시피 5: 비동기 스트리밍 에이전트

```python
import asyncio
from strands import Agent

async def main():
    agent = Agent()
    async for event in agent.stream_async("Tell me a story"):
        if "data" in event:
            print(event["data"], end="", flush=True)

asyncio.run(main())
```

## 일반적인 실수 방지

1. **도구 설명 부족** — LLM이 도구를 올바르게 선택하려면 명확한 docstring이 필요하다
2. **컨텍스트 오버플로우** — 긴 대화에는 SlidingWindow 또는 Summarizing 매니저를 사용한다
3. **MCP 컨텍스트 누락** — MCPClient는 Agent에 직접 전달하거나 `with` 블록 내에서 사용한다
4. **Cross-Region 에러** — Bedrock 모델 ID에 리전 접두사를 붙인다 (`us.anthropic.claude-sonnet-4-20250514-v1:0`)
5. **비동기 혼용** — 동기/비동기 패턴을 일관되게 사용한다. 비동기 도구는 `invoke_async()`로 호출한다

## 디버깅

```python
def debug_handler(**kwargs):
    if "data" in kwargs:
        print(f"[TEXT] {kwargs['data']}", end="")
    if "current_tool_use" in kwargs:
        tool = kwargs["current_tool_use"]
        if tool.get("name"):
            print(f"\n[TOOL] {tool['name']}")

agent = Agent(callback_handler=debug_handler)
```
