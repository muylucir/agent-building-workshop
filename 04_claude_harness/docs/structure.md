# Project Structure

## 프로젝트 루트 생성 규칙

명세서 파일이 있는 디렉토리에 직접 코드를 생성하지 않는다. 반드시 **별도의 프로젝트 디렉토리를 생성**하고 그 안에서 작업한다.

- 디렉토리 이름은 명세서의 에이전트 시스템 이름을 snake_case로 변환하여 사용
- 예: "수출 통관 자동화" → `export_customs_agent/`, "채용 이력서 평가" → `resume_evaluator/`

## 디렉토리 레이아웃

```
project-root/
├── agents/                    # 에이전트 클래스 (1 에이전트 = 1 파일)
│   ├── supervisor.py          # 멀티 에이전트: Supervisor/Orchestrator
│   ├── <agent_name>.py        # 각 Specialist 에이전트
│   └── ...
├── tools/                     # 도구 함수 (@tool 데코레이터)
│   ├── <domain>_tools.py      # 도메인별 도구 그룹
│   └── ...
├── mocks/                     # 도구의 mock 구현 (현실적 데이터)
│   ├── <domain>_mock.py       # 도메인별 mock 데이터 + 로직
│   └── data/                  # mock용 정적 데이터 파일 (JSON/CSV)
├── prompts/                   # 시스템 프롬프트 + 유저 프롬프트 템플릿
│   ├── <agent_name>_prompt.py # 에이전트별 프롬프트
│   └── ...
├── schemas/                   # Pydantic 모델 (입출력 검증)
│   └── models.py
├── config.py                  # 에이전트 프로필 (모델 ID, 토큰, 온도)
├── main.py                    # 실행 진입점 (CLI 또는 테스트용)
├── requirements.txt
└── README.md
```

## 파일 명명 규칙

| 대상 | 파일명 패턴 | 예시 |
|------|------------|------|
| 에이전트 클래스 | `agents/<snake_case_name>.py` | `agents/<agent_name>.py` |
| 도구 함수 | `tools/<domain>_tools.py` | `tools/<domain>_tools.py` |
| mock 구현 | `mocks/<domain>_mock.py` | `mocks/<domain>_mock.py` |
| mock 데이터 | `mocks/data/<name>.json` | `mocks/data/<domain>_records.json` |
| 프롬프트 | `prompts/<agent_name>_prompt.py` | `prompts/<agent_name>_prompt.py` |
| 스키마 | `schemas/models.py` 또는 `schemas/<domain>.py` | `schemas/models.py` |

## 명명 컨벤션

- 에이전트 클래스: `PascalCase` — `HSCodeClassifierAgent`, `SupervisorAgent`
- 도구 함수: `snake_case` — `@tool def tariff_schedule_retriever(...)`
- 프롬프트 상수: `UPPER_SNAKE_CASE` — `SUPERVISOR_SYSTEM_PROMPT`
- 프롬프트 생성 함수: `get_<name>_prompt()` — `get_supervisor_prompt(order_context)`

## 에이전트 클래스 구조

```python
# agents/<agent_name>.py
from strands import Agent
from config import get_profile

class SomeAgent:
    def __init__(self):
        cfg = get_profile("<agent_name>")
        self.agent = Agent(
            system_prompt=SYSTEM_PROMPT,
            model=model,
            tools=[...],
        )

    def run(self, input_data: dict) -> dict:
        """메인 실행 메서드"""
        ...
```

## config.py 구조

```python
import os

DEFAULT_MODEL = os.environ.get("AGENT_DEFAULT_MODEL", "us.anthropic.claude-sonnet-4-20250514-v1:0")

AGENT_PROFILES = {
    "<agent_name>": {
        "model_id": os.environ.get("AGENT_<NAME>_MODEL", DEFAULT_MODEL),
        "max_tokens": 8192,
        "temperature": 0.0,
    },
}

def get_profile(name: str) -> dict:
    return dict(AGENT_PROFILES[name])
```

## main.py 구조

main.py는 명세서의 Example User Prompt를 입력으로 받아 전체 파이프라인을 실행하는 진입점이다.

```python
# main.py
"""
<에이전트 시스템 이름> — 실행 진입점

사용법:
  python main.py                          # 기본 예시 입력으로 실행
  python main.py --input input.json       # JSON 파일 입력으로 실행
"""
import argparse
import json

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--input", type=str, help="입력 JSON 파일 경로")
    args = parser.parse_args()

    if args.input:
        with open(args.input) as f:
            input_data = json.load(f)
    else:
        # 명세서 Example User Prompt 기반 기본 입력
        input_data = { ... }

    # Agentic AI: Supervisor 에이전트 실행
    # AI-Assisted Workflow: Pipeline 오케스트레이터 실행
    result = run_pipeline(input_data)
    print(json.dumps(result, ensure_ascii=False, indent=2))

if __name__ == "__main__":
    main()
```

핵심 규칙:
- `--input` 옵션으로 JSON 파일을 받을 수 있어야 함
- 기본 입력은 명세서의 Example User Prompt 데이터를 하드코딩
- 결과를 JSON으로 stdout에 출력

## AI-Assisted Workflow 구조 변형

자율성 ≤5인 AI-Assisted Workflow 명세서의 경우 `agents/` 대신 `stages/`를 사용할 수 있다:

```
project-root/
├── stages/                    # 파이프라인 단계 (1 Stage = 1 파일)
│   ├── stage_1_event_receiver.py
│   ├── stage_2_resume_parser.py
│   └── ...
├── pipeline.py                # 파이프라인 오케스트레이터 (순차/병렬/조건 분기)
├── tools/
├── mocks/
├── prompts/
├── schemas/
├── config.py
├── main.py
└── requirements.txt
```
