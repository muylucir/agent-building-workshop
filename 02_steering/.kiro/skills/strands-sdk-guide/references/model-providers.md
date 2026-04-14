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
agent = Agent(model="global.anthropic.claude-sonnet-4-6")

# BedrockModel 인스턴스
bedrock = BedrockModel(
    model_id="global.anthropic.claude-sonnet-4-6",
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
  modelId: 'global.anthropic.claude-sonnet-4-6',
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
    model_id="global.anthropic.claude-sonnet-4-6",
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
    model_id="global.anthropic.claude-sonnet-4-6",
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
    model_id="global.anthropic.claude-sonnet-4-6",
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
    model_id="global.anthropic.claude-sonnet-4-6",
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


## 모델 전환

에이전트에서 쉽게 모델 전환:

```python
from strands import Agent
from strands.models import BedrockModel
from strands.models.openai import OpenAIModel

# Bedrock 사용
bedrock = BedrockModel(model_id="global.anthropic.claude-sonnet-4-6")
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
    model_id="global.anthropic.claude-sonnet-4-6",
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
model_id = "global.anthropic.claude-sonnet-4-6"
```

### 리전 해결 우선순위

1. `BedrockModel(region_name="...")` 명시적 지정
2. boto3 세션의 리전 (AWS_DEFAULT_REGION)
3. `AWS_REGION` 환경변수
4. 기본값 (us-west-2)
