# 대화 관리, 세션, Structured Output, Observability 가이드

## 목차
- [대화 관리](#대화-관리)
- [Structured Output](#structured-output)
- [세션 관리](#세션-관리)
- [관측성 (Observability)](#관측성-observability)

## 대화 관리

컨텍스트 윈도우를 효율적으로 관리하여 긴 대화에서도 안정적으로 동작하게 한다.

### NullConversationManager

대화 히스토리를 수정하지 않는다. 짧은 대화나 디버깅용.

```python
from strands import Agent
from strands.agent.conversation_manager import NullConversationManager

agent = Agent(conversation_manager=NullConversationManager())
```

### SlidingWindowConversationManager (기본값)

최근 N개 메시지만 유지한다.

```python
from strands.agent.conversation_manager import SlidingWindowConversationManager

manager = SlidingWindowConversationManager(
    window_size=20,
    should_truncate_results=True,
    per_turn=True
)

agent = Agent(conversation_manager=manager)
```

### SummarizingConversationManager

오래된 메시지를 요약하여 컨텍스트를 유지한다 (Python만).

```python
from strands.agent.conversation_manager import SummarizingConversationManager

manager = SummarizingConversationManager(
    summary_ratio=0.3,
    preserve_recent_messages=10
)

agent = Agent(conversation_manager=manager)
```

커스텀 요약 에이전트로 비용을 절감할 수 있다:

```python
from strands import Agent
from strands.models import AnthropicModel

summarization_model = AnthropicModel(
    model_id="claude-3-5-haiku-20241022",
    max_tokens=1000,
    params={"temperature": 0.1}
)
summarization_agent = Agent(model=summarization_model)

manager = SummarizingConversationManager(
    summarization_agent=summarization_agent
)
```

### TypeScript 대화 관리

```typescript
import { Agent, SlidingWindowConversationManager } from '@strands-agents/sdk'

const manager = new SlidingWindowConversationManager({
  windowSize: 40,
  shouldTruncateResults: true
})

const agent = new Agent({ conversationManager: manager })
```

## Structured Output

Pydantic 모델로 타입 안전한 응답을 받는다 (Python만).

### 기본 사용

```python
from pydantic import BaseModel, Field
from strands import Agent

class PersonInfo(BaseModel):
    name: str = Field(description="Name of the person")
    age: int = Field(description="Age of the person")
    occupation: str = Field(description="Occupation")

agent = Agent()
result = agent(
    "John Smith is a 30 year-old software engineer",
    structured_output_model=PersonInfo
)

person: PersonInfo = result.structured_output
print(f"Name: {person.name}")      # John Smith
print(f"Age: {person.age}")        # 30
print(f"Job: {person.occupation}") # software engineer
```

### 복잡한 스키마

```python
from pydantic import BaseModel, Field
from typing import List, Optional

class ProductAnalysis(BaseModel):
    name: str = Field(description="Product name")
    category: str = Field(description="Product category")
    price: float = Field(description="Price in USD")
    features: List[str] = Field(description="Key features")
    rating: Optional[float] = Field(description="Rating 1-5", ge=1, le=5)

result = agent(
    "UltraBook Pro is a $1,299 laptop with 4K display, 16GB RAM...",
    structured_output_model=ProductAnalysis
)
```

### 검증 자동 재시도

Pydantic validator가 실패하면 에이전트가 자동으로 재시도한다:

```python
from pydantic import BaseModel, field_validator

class Name(BaseModel):
    first_name: str

    @field_validator("first_name")
    @classmethod
    def validate_name(cls, value: str) -> str:
        if not value.endswith('abc'):
            raise ValueError("Name must end with 'abc'")
        return value

result = agent("Aaron's name", structured_output_model=Name)
```

### 도구와 함께 사용

```python
from strands_tools import calculator

class MathResult(BaseModel):
    operation: str = Field(description="The operation performed")
    result: int = Field(description="The result")

agent = Agent(tools=[calculator])
result = agent("What is 42 + 8", structured_output_model=MathResult)
```

### 에러 처리

```python
from strands.types.exceptions import StructuredOutputException

try:
    result = agent(prompt, structured_output_model=MyModel)
except StructuredOutputException as e:
    print(f"Structured output failed: {e}")
```

## 세션 관리

대화 상태를 지속적으로 저장하고 복원한다 (Python만).

### 파일 기반 세션

```python
from strands import Agent
from strands.agent.session_manager import FileSessionManager

session_manager = FileSessionManager(
    session_id="my-session",
    storage_dir="./sessions"
)

agent = Agent(session_manager=session_manager)
agent("My name is Alice")
# 세션 자동 저장

# 나중에 복원
agent2 = Agent(session_manager=FileSessionManager(
    session_id="my-session",
    storage_dir="./sessions"
))
response = agent2("What is my name?")  # Alice
```

### S3 세션

```python
from strands.agent.session_manager import S3SessionManager

session_manager = S3SessionManager(
    session_id="my-session",
    bucket_name="my-bucket",
    prefix="sessions/"
)

agent = Agent(session_manager=session_manager)
```

## 관측성 (Observability)

OpenTelemetry 통합으로 에이전트 실행을 모니터링한다 (Python만).

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import ConsoleSpanExporter, SimpleSpanProcessor

provider = TracerProvider()
provider.add_span_processor(SimpleSpanProcessor(ConsoleSpanExporter()))
trace.set_tracer_provider(provider)

from strands import Agent
agent = Agent()
result = agent("Hello")

# 메트릭 접근
print(f"Input tokens: {result.metrics.accumulated_usage.get('inputTokens')}")
print(f"Output tokens: {result.metrics.accumulated_usage.get('outputTokens')}")
```
