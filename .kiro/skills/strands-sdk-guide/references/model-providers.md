# 모델 프로바이더 가이드

## 목차
- [지원 프로바이더](#지원-프로바이더)
- [Amazon Bedrock](#amazon-bedrock)
- [OpenAI](#openai)
- [Anthropic](#anthropic)
- [기타 프로바이더](#기타-프로바이더)
- [커스텀 프로바이더](#커스텀-프로바이더)

## 지원 프로바이더

| 프로바이더 | Python | TypeScript |
|----------|--------|------------|
| Amazon Bedrock | O | O |
| OpenAI | O | O |
| Anthropic | O | X |
| Ollama | O | X |
| LiteLLM | O | X |
| Gemini | O | X |
| MistralAI | O | X |
| SageMaker | O | X |
| Custom | O | O |

## Amazon Bedrock

기본 프로바이더. Claude, Nova, Llama 등 다양한 모델 지원.

### 기본 사용

```python
from strands import Agent
from strands.models import BedrockModel

# 기본값 (Claude Sonnet 4)
agent = Agent()

# 모델 ID 직접 지정
agent = Agent(model="us.anthropic.claude-sonnet-4-20250514-v1:0")

# BedrockModel 인스턴스
bedrock = BedrockModel(
    model_id="us.anthropic.claude-sonnet-4-20250514-v1:0",
    temperature=0.3,
    max_tokens=4096
)
agent = Agent(model=bedrock)
```

### TypeScript

```typescript
import { Agent } from '@strands-agents/sdk'
import { BedrockModel } from '@strands-agents/sdk/bedrock'

const bedrock = new BedrockModel({
  modelId: 'us.anthropic.claude-sonnet-4-20250514-v1:0',
  temperature: 0.3,
  maxTokens: 4096
})

const agent = new Agent({ model: bedrock })
```

### 상세 설정

```python
from strands.models import BedrockModel
from botocore.config import Config as BotocoreConfig

boto_config = BotocoreConfig(
    retries={"max_attempts": 3, "mode": "standard"},
    connect_timeout=5,
    read_timeout=60
)

bedrock = BedrockModel(
    model_id="us.anthropic.claude-sonnet-4-20250514-v1:0",
    region_name="us-east-1",
    temperature=0.3,
    top_p=0.8,
    stop_sequences=["###", "END"],
    boto_client_config=boto_config
)
```

### 가드레일

```python
bedrock = BedrockModel(
    model_id="us.anthropic.claude-sonnet-4-20250514-v1:0",
    guardrail_id="your-guardrail-id",
    guardrail_version="DRAFT",
    guardrail_trace="enabled",
    guardrail_redact_input=True,
    guardrail_redact_output=False
)
```

### 캐싱

프롬프트, 도구, 메시지 캐싱으로 비용 절감:

```python
from strands import Agent
from strands.types.content import SystemContentBlock

# 시스템 프롬프트 캐싱
system_content = [
    SystemContentBlock(text="긴 시스템 프롬프트..." * 500),
    SystemContentBlock(cachePoint={"type": "default"})
]

agent = Agent(system_prompt=system_content)

# 도구 캐싱
from strands.models import BedrockModel

bedrock = BedrockModel(
    model_id="us.anthropic.claude-sonnet-4-20250514-v1:0",
    cache_tools="default"
)
agent = Agent(model=bedrock, tools=[tool1, tool2])
```

### 메시지 캐싱

```python
messages = [
    {
        "role": "user",
        "content": [
            {"document": {"format": "txt", "name": "doc", "source": {"bytes": b"..."}}},
            {"text": "Use this document"},
            {"cachePoint": {"type": "default"}}
        ]
    },
    {
        "role": "assistant",
        "content": [{"text": "I will reference that document."}]
    }
]

agent = Agent(messages=messages)
```

### 멀티모달 입력

```python
response = agent([
    {
        "document": {
            "format": "txt",
            "name": "example",
            "source": {"bytes": b"Document content..."}
        }
    },
    {"text": "Summarize this document."}
])
```

### 추론(Reasoning) 모드

```python
bedrock = BedrockModel(
    model_id="us.anthropic.claude-sonnet-4-20250514-v1:0",
    additional_request_fields={
        "thinking": {
            "type": "enabled",
            "budget_tokens": 4096
        }
    }
)
```

### 런타임 설정 변경

```python
bedrock = BedrockModel(model_id="...", temperature=0.7)

# 나중에 설정 변경
bedrock.update_config(temperature=0.3, top_p=0.8)
```

## OpenAI

GPT 모델 지원.

### Python

```python
from strands import Agent
from strands.models.openai import OpenAIModel

openai_model = OpenAIModel(
    client_args={"api_key": "your-api-key"},
    model_id="gpt-4o"
)

agent = Agent(model=openai_model)
```

### TypeScript

```typescript
import { Agent } from '@strands-agents/sdk'
import { OpenAIModel } from '@strands-agents/sdk/openai'

const openai = new OpenAIModel({
  apiKey: process.env.OPENAI_API_KEY,
  modelId: 'gpt-4o'
})

const agent = new Agent({ model: openai })
```

## Anthropic

Claude 모델 직접 API 연결 (Python만 지원).

```python
from strands import Agent
from strands.models import AnthropicModel

anthropic_model = AnthropicModel(
    model_id="claude-sonnet-4-20250514",
    max_tokens=4096,
    params={"temperature": 0.3}
)

agent = Agent(model=anthropic_model)
```

## 기타 프로바이더

### Ollama (로컬 모델)

```python
from strands.models.ollama import OllamaModel

ollama = OllamaModel(
    host="http://localhost:11434",
    model_id="llama3.2"
)
agent = Agent(model=ollama)
```

### LiteLLM (통합 인터페이스)

```python
from strands.models.litellm import LiteLLMModel

litellm = LiteLLMModel(
    model_id="gpt-4o",
    params={"api_key": "your-key"}
)
agent = Agent(model=litellm)
```

### Gemini

```python
from strands.models.gemini import GeminiModel

gemini = GeminiModel(
    model_id="gemini-pro",
    client_args={"api_key": "your-key"}
)
agent = Agent(model=gemini)
```

### MistralAI

```python
from strands.models.mistral import MistralModel

mistral = MistralModel(
    model_id="mistral-large-latest",
    client_args={"api_key": "your-key"}
)
agent = Agent(model=mistral)
```

## 커스텀 프로바이더

자체 모델 프로바이더 구현:

### Python

```python
from strands.models import Model
from strands.types.models import ModelConfig
from typing import Any, Generator

class CustomModel(Model):
    def __init__(self, api_key: str, model_id: str):
        self.api_key = api_key
        self.model_id = model_id

    def get_config(self) -> ModelConfig:
        return ModelConfig(
            model_id=self.model_id,
            max_tokens=4096,
            temperature=0.7
        )

    def update_config(self, **kwargs):
        # 설정 업데이트 로직
        pass

    def format_request(self, messages: list, tools: list, system_prompt: str) -> dict:
        # 요청 포맷팅
        return {
            "messages": messages,
            "tools": tools,
            "system": system_prompt
        }

    def stream(self, request: dict) -> Generator[dict, None, None]:
        # 스트리밍 응답 생성
        response = self._call_api(request)
        for chunk in response:
            yield self._format_chunk(chunk)

    def _call_api(self, request: dict) -> Any:
        # API 호출 구현
        pass

    def _format_chunk(self, chunk: Any) -> dict:
        # 청크 포맷팅
        pass

# 사용
custom = CustomModel(api_key="...", model_id="custom-model")
agent = Agent(model=custom)
```

### TypeScript

```typescript
import { Model, ModelConfig, Message, ToolSpec } from '@strands-agents/sdk'

class CustomModel implements Model {
  private apiKey: string
  private modelId: string

  constructor(apiKey: string, modelId: string) {
    this.apiKey = apiKey
    this.modelId = modelId
  }

  getConfig(): ModelConfig {
    return {
      modelId: this.modelId,
      maxTokens: 4096,
      temperature: 0.7
    }
  }

  updateConfig(config: Partial<ModelConfig>): void {
    // 설정 업데이트
  }

  formatRequest(messages: Message[], tools: ToolSpec[], systemPrompt: string) {
    return { messages, tools, system: systemPrompt }
  }

  async *stream(request: any): AsyncGenerator<any> {
    // 스트리밍 구현
  }
}
```

## 모델 전환

에이전트에서 쉽게 모델 전환:

```python
from strands import Agent
from strands.models import BedrockModel
from strands.models.openai import OpenAIModel

# Bedrock 사용
bedrock = BedrockModel(model_id="us.anthropic.claude-sonnet-4-20250514-v1:0")
agent = Agent(model=bedrock)
response1 = agent("Hello from Bedrock")

# OpenAI로 전환
openai = OpenAIModel(client_args={"api_key": "..."}, model_id="gpt-4o")
agent = Agent(model=openai)
response2 = agent("Hello from OpenAI")
```

## 스트리밍 vs 비스트리밍

일부 모델은 스트리밍 도구 사용을 지원하지 않는다:

```python
# 스트리밍 (기본값)
streaming_model = BedrockModel(
    model_id="us.anthropic.claude-sonnet-4-20250514-v1:0",
    streaming=True
)

# 비스트리밍
non_streaming_model = BedrockModel(
    model_id="us.meta.llama3-2-90b-instruct-v1:0",
    streaming=False
)
```

## 트러블슈팅

### Cross-Region Inference 에러

```python
# 잘못됨
model_id = "anthropic.claude-sonnet-4-20250514-v1:0"

# 올바름 - 리전 접두사 추가
model_id = "us.anthropic.claude-sonnet-4-20250514-v1:0"
```

### 리전 해결 우선순위

1. `BedrockModel(region_name="...")` 명시적 지정
2. boto3 세션의 리전 (AWS_DEFAULT_REGION)
3. `AWS_REGION` 환경변수
4. 기본값 (us-west-2)
