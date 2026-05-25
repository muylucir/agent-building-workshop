# Technology Stack

## 필수 기술 스택

| 영역 | 기술 | 비고 |
|------|------|------|
| 언어 | **Python 3.12** | |
| Agent 프레임워크 | **Strands Agents SDK** | `pip install strands-agents strands-agents-tools` |
| LLM | AWS Bedrock Claude 계열 | BedrockModel 사용 |
| 상태 저장 | 인메모리 dict 또는 Pydantic 모델 | 프로토타입 단계 |

## 금지 사항

- LangChain, LangGraph, CrewAI, OpenAI Assistants 등 다른 에이전트 프레임워크 사용 금지
- raw boto3로 Bedrock API 직접 호출 금지 — 반드시 Strands SDK의 BedrockModel 사용
- OpenAI, Anthropic Direct API 사용 금지 — 반드시 AWS Bedrock 경유

## 코드 언어 정책

- 코드 주석: **영어**
- 변수명/함수명/클래스명: **영어**
- README.md: **한국어** (명세서가 한국어이므로)
- 시스템 프롬프트: **명세서 원문 그대로** (한국어/영어 혼용 가능 — 명세서에 따름)

## Skill 참조 안내

SDK 사용법의 상세 가이드는 별도 skill에 정의되어 있다:

- **Strands SDK 사용법** → `strands-sdk-guide` skill 참조
  - Agent 생성, @tool 데코레이터, MCP 연동, 멀티에이전트(Swarm/Graph/Workflow), hooks, 스트리밍, 세션 관리, Structured Output

## 모델 설정 기본값

```python
from strands.models import BedrockModel

model = BedrockModel(
    model_id="global.anthropic.claude-sonnet-4-6",  # 리전 접두사 필수
    max_tokens=8192,
    temperature=0.0,
)
```

- 모델 ID는 명세서의 요구사항 또는 비용/성능 트레이드오프에 따라 조정
- `temperature=0.0`을 기본으로 하되, 창의적 생성이 필요한 에이전트는 명세서 지시에 따라 조정

## 도구 모킹 전략

→ **`mock.md`** 참조 — mock 구현 원칙, 패턴, 데이터 소스 우선순위, 필수 규칙 등

## 실행 방법

### 환경 설정

```bash
# uv 설치 (미설치 시)
curl -LsSf https://astral.sh/uv/install.sh | sh

# 프로젝트 디렉토리에서 의존성 설치
cd <project-root>
uv venv --python 3.12
source .venv/bin/activate
uv pip install -r requirements.txt
```

### AWS 인증

Instance Profile을 사용한다. 별도의 AWS 자격증명 파일이나 환경변수 설정이 필요 없다.

- EC2 인스턴스에 Bedrock 접근 권한이 포함된 IAM Role이 연결되어 있어야 함
- 필요 권한: `bedrock:InvokeModel`, `bedrock:InvokeModelWithResponseStream`

### 실행

```bash
# 기본 예시 입력으로 실행
python main.py

# JSON 파일 입력으로 실행
python main.py --input input.json
```