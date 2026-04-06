# Hooks & 스트리밍 가이드

## 목차
- [Hooks](#hooks)
- [스트리밍](#스트리밍)

## Hooks

에이전트 라이프사이클 이벤트에 콜백을 등록하여 동작을 확장한다. 모니터링, 도구 인터셉션, 검증, 메트릭 수집에 활용한다.

### 이벤트 타입

| 이벤트 | 설명 |
|-------|------|
| `AgentInitializedEvent` | 에이전트 초기화 완료 |
| `BeforeInvocationEvent` | 에이전트 호출 시작 |
| `AfterInvocationEvent` | 에이전트 호출 완료 |
| `MessageAddedEvent` | 메시지 추가됨 |
| `BeforeModelCallEvent` | 모델 호출 전 |
| `AfterModelCallEvent` | 모델 호출 후 |
| `BeforeToolCallEvent` | 도구 호출 전 |
| `AfterToolCallEvent` | 도구 호출 후 |

### 개별 콜백 등록

```python
from strands import Agent
from strands.hooks import BeforeInvocationEvent, BeforeToolCallEvent

agent = Agent()

def log_start(event: BeforeInvocationEvent):
    print(f"Starting invocation for agent: {event.agent.name}")

def log_tool(event: BeforeToolCallEvent):
    print(f"Calling tool: {event.tool_use['name']}")

agent.hooks.add_callback(BeforeInvocationEvent, log_start)
agent.hooks.add_callback(BeforeToolCallEvent, log_tool)
```

### HookProvider 패턴

여러 훅을 하나의 클래스로 묶어 재사용하려면 HookProvider를 상속한다.

```python
from strands.hooks import HookProvider, HookRegistry
from strands.hooks import BeforeInvocationEvent, AfterInvocationEvent, BeforeToolCallEvent

class LoggingHook(HookProvider):
    def register_hooks(self, registry: HookRegistry) -> None:
        registry.add_callback(BeforeInvocationEvent, self.log_start)
        registry.add_callback(AfterInvocationEvent, self.log_end)
        registry.add_callback(BeforeToolCallEvent, self.log_tool)

    def log_start(self, event: BeforeInvocationEvent):
        print(f"[START] Agent: {event.agent.name}")

    def log_end(self, event: AfterInvocationEvent):
        print(f"[END] Agent: {event.agent.name}")

    def log_tool(self, event: BeforeToolCallEvent):
        print(f"[TOOL] {event.tool_use['name']}")

agent = Agent(hooks=[LoggingHook()])
```

### TypeScript Hooks

```typescript
import { Agent, HookProvider, HookRegistry } from '@strands-agents/sdk'
import { BeforeInvocationEvent, AfterInvocationEvent } from '@strands-agents/sdk'

class LoggingHook implements HookProvider {
  registerCallbacks(registry: HookRegistry): void {
    registry.addCallback(BeforeInvocationEvent, (ev) => this.logStart(ev))
    registry.addCallback(AfterInvocationEvent, (ev) => this.logEnd(ev))
  }

  private logStart(event: BeforeInvocationEvent): void {
    console.log('Request started')
  }

  private logEnd(event: AfterInvocationEvent): void {
    console.log('Request completed')
  }
}

const agent = new Agent({ hooks: [new LoggingHook()] })
```

### 도구 인터셉션

BeforeToolCallEvent에서 도구를 교체하거나 취소할 수 있다.

```python
class ToolInterceptor(HookProvider):
    def register_hooks(self, registry: HookRegistry) -> None:
        registry.add_callback(BeforeToolCallEvent, self.intercept)

    def intercept(self, event: BeforeToolCallEvent):
        # 도구 교체
        if event.tool_use["name"] == "sensitive_tool":
            event.selected_tool = self.safe_alternative
            event.tool_use["name"] = "safe_tool"

        # 도구 취소
        if event.tool_use["name"] == "blocked_tool":
            event.cancel_tool = "This tool is not allowed"
```

### 도구 호출 제한

```python
class LimitToolCounts(HookProvider):
    def __init__(self, max_counts: dict[str, int]):
        self.max_counts = max_counts
        self.counts = {}

    def register_hooks(self, registry: HookRegistry) -> None:
        registry.add_callback(BeforeInvocationEvent, self.reset)
        registry.add_callback(BeforeToolCallEvent, self.check)

    def reset(self, event: BeforeInvocationEvent):
        self.counts = {}

    def check(self, event: BeforeToolCallEvent):
        name = event.tool_use["name"]
        self.counts[name] = self.counts.get(name, 0) + 1
        max_count = self.max_counts.get(name)
        if max_count and self.counts[name] > max_count:
            event.cancel_tool = f"Tool '{name}' limit exceeded"

agent = Agent(hooks=[LimitToolCounts({"api_call": 5})])
```

### Invocation State 접근

```python
def log_with_context(event: BeforeToolCallEvent):
    user_id = event.invocation_state.get("user_id", "unknown")
    print(f"User {user_id} calling: {event.tool_use['name']}")

agent.hooks.add_callback(BeforeToolCallEvent, log_with_context)
result = agent("Do something", user_id="user123", session_id="sess456")
```

## 스트리밍

실시간으로 에이전트 이벤트를 처리한다.

### Python 비동기 이터레이터

```python
import asyncio
from strands import Agent

async def stream_response():
    agent = Agent()

    async for event in agent.stream_async("Tell me a story"):
        if "data" in event:
            print(event["data"], end="", flush=True)
        if "current_tool_use" in event:
            tool = event["current_tool_use"]
            if tool.get("name"):
                print(f"\n[Tool: {tool['name']}]")
        if "result" in event:
            print("\n--- Done ---")

asyncio.run(stream_response())
```

### Python 콜백 핸들러

```python
from strands import Agent

def my_handler(**kwargs):
    if "data" in kwargs:
        print(kwargs["data"], end="")
    if "current_tool_use" in kwargs:
        tool = kwargs["current_tool_use"]
        if tool.get("name"):
            print(f"\n[Tool: {tool['name']}]")

agent = Agent(callback_handler=my_handler)
agent("Calculate 2 + 2")
```

### TypeScript 스트리밍

```typescript
import { Agent } from '@strands-agents/sdk'

const agent = new Agent()

for await (const event of agent.stream('Tell me a story')) {
  switch (event.type) {
    case 'modelContentBlockDeltaEvent':
      if (event.delta.type === 'textDelta') {
        process.stdout.write(event.delta.text)
      }
      break
    case 'modelContentBlockStartEvent':
      if (event.start?.type === 'toolUseStart') {
        console.log(`\n[Tool: ${event.start.name}]`)
      }
      break
    case 'afterInvocationEvent':
      console.log('\nDone!')
      break
  }
}
```

### 이벤트 타입 요약

**라이프사이클**: `init_event_loop`, `start_event_loop`, `message`, `result`
**모델 스트림**: `data` (텍스트 청크), `delta` (원시 콘텐츠), `reasoning` (추론)
**도구**: `current_tool_use` (도구 정보), `tool_stream_event` (도구 스트리밍)
