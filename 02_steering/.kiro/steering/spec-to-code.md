# Spec-to-Code Mapping Rules

PATH 명세서의 각 섹션을 코드로 변환할 때 따라야 할 규칙.

## 1. Executive Summary → 프로젝트 초기화

- `Automation Level`을 확인하여 프로젝트 구조 결정:
  - **Agentic AI** → `agents/` 디렉토리 구조, Strands 멀티에이전트 패턴 사용
  - **AI-Assisted Workflow** → `stages/` + `pipeline.py` 구조
- `Solution`의 3계층 패턴 조합을 `README.md`에 기록
- `Next Steps`는 참고용으로만 확인 — **구현 범위는 명세서에 정의된 모든 에이전트·도구·상태·에러처리를 전체 구현하는 것**

## 2. Agent Components → 에이전트 클래스

명세서의 Agent Components 테이블(또는 워크플로우 단계 테이블)의 각 행이 하나의 에이전트 클래스(또는 Stage 클래스)에 대응한다.

### Agentic AI 매핑

| 명세서 필드 | 코드 대응 |
|------------|----------|
| Agent Name | 클래스명 (`agents/<snake_case>.py`) |
| Role | 클래스 docstring |
| Input | `run()` 메서드 파라미터 |
| Output | `run()` 메서드 반환값 |
| 인지 패턴 | Agent 구성 방식 결정 (아래 참조) |
| 도구 목록 | `tools=[...]` 파라미터 |
| 판단 로직 | 시스템 프롬프트의 `<instructions>` 또는 코드 로직 |

### AI-Assisted Workflow 매핑

| 명세서 필드 | 코드 대응 |
|------------|----------|
| 워크플로우 단계 | Stage 클래스 (`stages/stage_N_<name>.py`) |
| 처리 방식 | LLM 호출 여부 결정 — "결정적 로직"이면 AI 없이 순수 Python, "AI Step"이면 Strands Agent 사용 |
| 도구 목록 | Stage 내부에서 사용하는 함수/API 클라이언트 |
| 판단 로직 | `pipeline.py`의 조건 분기 또는 Stage 내부 로직 |

### 인지 패턴 → Strands 구성

| 명세서 인지 패턴 | Strands 구현 |
|-----------------|-------------|
| ReAct | 기본 Agent loop (도구 제공하면 자동 ReAct) |
| RAG | 검색 도구를 Agent에 제공 |
| Reflection | Agent 호출 → 결과를 Critic Agent에 전달 → 재생성 루프 |
| Planning | 시스템 프롬프트에 계획 수립 지시 포함 |
| Prompt Chaining | 순차적 Agent 호출 체인 (이전 출력 → 다음 입력) |
| Routing | 시스템 프롬프트에 분류 지시 + 조건 분기 코드 |
| Parallelization | `asyncio.gather()` 또는 `concurrent.futures`로 병렬 실행 |
| Human-in-the-Loop | 체크포인트에서 파이프라인 일시 정지 + 외부 입력 대기 |

## 3. Agent Prompts → 프롬프트 모듈

명세서의 각 에이전트 프롬프트 섹션을 `prompts/<agent_name>_prompt.py`로 변환한다.

```python
# prompts/<agent_name>_prompt.py

SYSTEM_PROMPT = """<role>
...명세서의 System Prompt 그대로 사용...
</role>
...
"""

def get_user_prompt(input_data: dict) -> str:
    """명세서의 Example User Prompt 구조를 템플릿화"""
    return f"""{{
  "order_id": "{input_data['order_id']}",
  ...
}}"""
```

규칙:
- System Prompt는 명세서에 작성된 내용을 **그대로 복사**하여 사용 — 수정, 요약, 재작성 금지
- Example User Prompt → `get_user_prompt()` 함수로 템플릿화하여 동적 데이터 삽입 (프롬프트 텍스트 자체는 변경하지 않고 변수 부분만 치환)

## 4. Tool Definitions → @tool 함수 + mock 구현

명세서의 Compact Signature를 Strands `@tool` 데코레이터 함수로 변환하되, 실제 API 대신 **현실적인 mock**으로 구현한다.

### 변환 규칙

명세서 형식:
```
- Purpose: 관세율표 DB를 RAG로 검색하여 후보 HS코드 목록을 반환
- Signature: tariff_schedule_retriever(item_description: str, top_k?: int = 10) -> list[{hs_code: str, description: str}]
- When to use: HS Code Classifier Agent가 분류 후보 코드를 탐색할 때
```

변환 결과 — 도구 함수 (tools/):
```python
from strands import tool
from mocks.<domain>_mock import mock_<tool_name>

@tool
def <tool_name>(query: str, top_k: int = 10) -> list:
    """<Purpose 그대로 사용>

    Args:
        query: <Signature 파라미터 설명>
        top_k: 반환할 최대 후보 수
    """
    return mock_<tool_name>(query, top_k)
```

변환 결과 — mock 구현 (mocks/):
```python
# mocks/<domain>_mock.py
import json
from pathlib import Path

_DATA_DIR = Path(__file__).parent / "data"
_RECORDS = json.loads((_DATA_DIR / "<domain>_records.json").read_text())

def mock_<tool_name>(query: str, top_k: int = 10) -> list:
    """키워드 매칭 기반 mock — 입력에 따라 다른 결과 반환"""
    query_lower = query.lower()
    results = []
    for entry in _RECORDS:
        score = sum(1 for kw in entry["keywords"] if kw in query_lower)
        if score > 0:
            results.append({**entry, "similarity_score": round(score / len(entry["keywords"]), 2)})
    return sorted(results, key=lambda x: x["similarity_score"], reverse=True)[:top_k]
```

### mock 데이터 파일 (mocks/data/):
```json
// mocks/data/<domain>_records.json — 명세서 <examples> 블록에서 추출
[
  {
    "id": "...",
    "description": "...",
    "keywords": ["keyword1", "keyword2"]
  }
]
```

### mock 구현 필수 규칙

1. **도구 함수(`tools/`)와 mock 구현(`mocks/`)을 분리** — 나중에 실제 API로 교체할 때 도구 함수의 import만 변경
2. **명세서 `<examples>` 블록의 입출력 데이터를 mock 데이터의 기준으로 사용** — 명세서 예시가 mock을 통과하면 에이전트가 정상 동작하는 것
3. **명세서 `<output_format>`의 JSON 구조를 정확히 준수** — mock 반환값이 명세서 출력 스키마와 일치해야 함
4. **입력에 따라 결과가 달라지도록 구현** — 키워드 매칭, 조건 분기, 룩업 테이블 등 활용
5. **에러 시뮬레이션 지원** — 특정 입력값(예: `"__TIMEOUT__"`)으로 타임아웃/오류를 트리거할 수 있도록 구현

## 5. State Management → 공유 상태

### Shared State

명세서의 Shared State 테이블 → Pydantic 모델로 구현:

```python
# schemas/models.py
from pydantic import BaseModel

class PipelineState(BaseModel):
    """명세서 Shared State 테이블의 모든 필드를 Pydantic 모델로 정의"""
    pipeline_id: str
    # ... 명세서 Shared State 테이블의 각 필드를 여기에 추가
    # 예: <field_name>: <type> | None = None
```

### Session State

세션 내 유지 데이터 → 에이전트 인스턴스 속성 또는 세션 딕셔너리로 관리

### Persistent State

영구 저장 데이터 → 프로토타입에서는 JSON 파일 또는 인메모리 dict로 mock

## 6. Error Handling → Fallback 로직

명세서의 Error Handling 테이블을 코드로 변환한다.

| 명세서 컬럼 | 코드 대응 |
|------------|----------|
| 오류 시나리오 | try/except 블록의 대상 |
| 감지 조건 | except 조건 또는 응답 검증 로직 |
| 복구 전략 | 재시도 로직 (지수 백오프 등) |
| 최종 Fallback | 재시도 실패 후 대체 경로 |

공통 패턴:
```python
import time

def retry_with_backoff(fn, max_retries=3, base_delay=1):
    for attempt in range(max_retries):
        try:
            return fn()
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(base_delay * (2 ** attempt))
```

## 7. Layer3 Agentic Workflow → 멀티에이전트 패턴

명세서의 Layer3 패턴에 따라 Strands 멀티에이전트 구현 방식을 선택한다. 상세 API는 strands-sdk-guide skill의 multi-agent 문서를 참조.

| Layer3 패턴 | Strands 구현 |
|------------|-------------|
| Agents as Tools | Supervisor Agent가 Specialist Agent를 도구처럼 호출 |
| Swarm | `from strands.multiagent import Swarm` — 에이전트 간 핸드오프 |
| Graph | `from strands.multiagent import GraphAgent` — 방향성 그래프 흐름 |
| Workflow | 순차 파이프라인 — Prompt Chaining 또는 코드 오케스트레이션 |
| 싱글 에이전트 | 단일 Agent + tools 구성 |

## 구현 순서 권장

1. `config.py` — 에이전트 프로필 정의
2. `schemas/models.py` — 공유 상태 모델 정의
3. `mocks/data/` — 명세서 예시 기반 mock 데이터 파일 생성
4. `mocks/` — mock 구현 (키워드 매칭, 룩업 등)
5. `prompts/` — 명세서 프롬프트를 모듈로 변환
6. `tools/` — 도구 함수 (mock import 연결)
7. `agents/` 또는 `stages/` — 에이전트/스테이지 클래스 구현
8. `main.py` — 실행 진입점
9. E2E 테스트 — 명세서 Example User Prompt로 파이프라인 검증
