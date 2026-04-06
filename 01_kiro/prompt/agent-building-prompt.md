# AI Agent 시스템 개발 요청

## 입력 정보

- **명세서 파일**: {명세서 파일명}
- **Bedrock Model ID**: global.anthropic.claude-sonnet-4-6

---

## 1단계: 명세서 분석 (구현 전 필수)

위에 지정한 명세서 파일을 읽고 아래 항목을 추출하여 구현 계획을 먼저 정리하세요.
파일을 읽기 전에 임의로 구조를 가정하지 마세요. 항상 명세서상의 모든 Phase를 구현하세요.

### 추출할 항목

| 추출 항목 | 찾아야 할 내용 | 추출 결과 활용처 |
|-----------|---------------|-----------------|
| 에이전트 목록 (이름, 역할, 인지 패턴, 도구 목록) | 에이전트 컴포넌트/구성요소 정의 섹션 | `src/agents/` 디렉토리 구성 |
| 각 에이전트의 시스템 프롬프트 원본 | 에이전트 프롬프트 정의 섹션 | `src/prompts/` 파일에 원본 저장 |
| 도구 전체 목록 (이름, 시그니처, Purpose, When to use) | 도구(Tool) 정의 섹션 | `src/tools/` 구현 및 Mock 설계 |
| State 필드 (Shared / Session / Persistent 등) | 상태 관리 정의 섹션 | `src/models/state.py` 정의 |
| 파이프라인 실행 흐름 (병렬/순차 구간) | 워크플로우/시퀀스 다이어그램 섹션 | Orchestrator 흐름 구현 |
| 오류 시나리오 및 복구 전략 | 오류 처리/Fallback 전략 섹션 (있는 경우) | 각 도구/에이전트 try/except 설계 |
| Human-in-the-Loop 조건 및 라우팅 규칙 | HiL 조건이 정의된 모든 섹션 (있는 경우) | 신뢰도 판정 로직 구현 |

추출이 완료되면 다음 형식으로 요약 출력 후 구현을 시작하세요:

```
[명세서 분석 결과]
- 에이전트 수: N개 (오케스트레이터 포함)
- 에이전트 목록: A, B, C, ...
- 도구 수: N개
- 병렬 실행 구간: ...
- HiL 조건: ...
```

---

## 2단계: 기술 스택

- **언어**: Python 3.12
- **에이전트 프레임워크**: Strands Agents SDK (Python)
  → SDK 사용법은 `strands-sdk-guide` skill을 반드시 먼저 확인하고 따르세요
  → SDK 사용법을 추측하거나 임의로 작성하지 마세요
- **LLM**: Amazon Bedrock
  → 모든 에이전트에 동일하게 global.anthropic.claude-sonnet-4-6 사용
- **외부 API**: 전부 Mock으로 시작, `USE_MOCK` 플래그로 Real과 전환
  → Bedrock LLM 호출은 Mock 대상이 아님 — 항상 실제 Bedrock을 사용

---

## 3단계: 핵심 원칙

### 원칙 1 — 명세서 프롬프트 원본 사용 (절대 수정 금지)

명세서의 프롬프트 정의 섹션에 있는 시스템 프롬프트를 **한 글자도 수정하지 않고** 그대로 사용합니다.
요약, 번역, 개선, 재구성 모두 금지입니다.

- 각 프롬프트를 `src/prompts/{agent_name}.py` 파일에 문자열 상수로 저장하세요
- 파일 최상단에 `# ⚠️ 명세서 원본 — 수정 금지` 주석을 붙이세요
- 저장한 상수를 에이전트 생성 시 system_prompt에 그대로 전달하세요

### 원칙 2 — Mock-First 전략

명세서의 도구 정의 섹션에 있는 도구 중 **외부 API를 호출하는 도구**를 Mock으로 구현합니다.
Bedrock LLM 호출(에이전트 추론)은 Mock 대상이 아니며 항상 실제 Bedrock을 사용합니다.

Mock 구현 규칙:
- Mock 응답 데이터는 명세서의 출력 형식 정의(output_format 등)와 예시를 기반으로 작성
- Mock과 Real API의 **함수 시그니처(파라미터명, 타입, 반환 구조)는 반드시 동일**하게 유지
- `src/config.py`의 `USE_MOCK: bool` 플래그로 Mock ↔ Real 전환
- Real 구현 자리는 `raise NotImplementedError("Real API not yet implemented")` 유지
- Mock 데이터는 `src/mock/` 디렉토리에 도메인별로 분리하여 관리

### 원칙 3 — Strands SDK 패턴 준수

`strands-sdk-guide` skill에서 다음 항목을 확인하고 해당 패턴을 그대로 따르세요:
- Agent 인스턴스 생성 및 system_prompt, tools 설정
- `@tool` 데코레이터를 사용한 커스텀 도구 정의와 docstring 작성 규칙
- Amazon Bedrock 모델 연결 방법 (BedrockModel 또는 해당 프로바이더 클래스)
- **Agents as Tools 패턴**: 서브에이전트를 도구로 래핑하여 오케스트레이터에 등록하는 방법
- 병렬 도구 호출이 SDK에서 처리되는 방식

### 원칙 4 — 명세서 설계 충실도

명세서에 정의된 모든 설계를 코드에 반영하세요:
- 에이전트별 도구 할당 (에이전트 컴포넌트 섹션에 정의된 도구 목록)
- 상태 관리 계층 구조 (명세서 상태 관리 섹션)
- 병렬/순차 실행 구간 (워크플로우/시퀀스 다이어그램 섹션)
- 모든 오류 시나리오의 복구 전략 (오류 처리 섹션, 없으면 생략)
- HiL 라우팅 규칙을 코드 로직으로 구현 — LLM이 아닌 결정론적 규칙으로 (HiL 관련 섹션, 없으면 생략)

---

## 4단계: 프로젝트 구조

명세서 분석 결과를 바탕으로 아래 템플릿에서 `{...}` 부분을 채워 실제 구조를 생성하세요.

```
{PROJECT_NAME}/
├── pyproject.toml
├── .env.example
├── README.md
│
├── src/
│   ├── __init__.py
│   ├── config.py                        # USE_MOCK, Bedrock 설정, 환경변수
│   │
│   ├── models/                          # Pydantic 데이터 모델
│   │   ├── __init__.py
│   │   ├── state.py                     # SharedState, SessionState, PersistentState
│   │   └── {domain_models}.py           # 명세서 입출력 기반 도메인 모델
│   │
│   ├── prompts/                         # 명세서 원본 프롬프트 (수정 금지)
│   │   ├── __init__.py
│   │   └── {agent_name}.py              # 에이전트별 1파일, 원본 그대로 저장
│   │
│   ├── tools/                           # @tool 데코레이터 기반 도구
│   │   ├── __init__.py
│   │   ├── {tool_group}.py              # 명세서 도구를 도메인별로 묶어 파일 분리
│   │   └── common.py                    # 다수 에이전트가 공유하는 도구
│   │
│   ├── mock/                            # Mock 구현
│   │   ├── __init__.py
│   │   ├── mock_data.py                 # 명세서 예시 기반 Mock 응답 데이터 상수
│   │   └── mock_{domain}.py             # 도메인별 Mock 함수
│   │
│   ├── agents/                          # Strands Agent 정의
│   │   ├── __init__.py
│   │   └── {agent_name}.py              # 에이전트별 1파일
│   │
│   ├── state/                           # 상태 관리
│   │   ├── __init__.py
│   │   ├── shared_state.py
│   │   ├── session_state.py
│   │   └── persistent_state.py          # SQLite 또는 JSON 기반
│   │
│   └── main.py                          # 진입점
│
└── tests/
    ├── __init__.py
    ├── test_tools/                      # 도구 단위 테스트
    ├── test_agents/                     # 에이전트 단위 테스트
    ├── test_e2e/
    │   ├── test_normal_flow.py          # 명세서 Example 기반
    │   └── test_edge_cases.py           # 명세서 Edge Case 기반
    └── fixtures/                        # 명세서 Example Input/Output을 그대로 저장하여 테스트 입력으로 사용
```

파일/디렉토리 이름의 `{...}` 부분은 명세서에서 추출한 에이전트명, 도메인명 등으로 채우세요.

---

## 5단계: 구현 규칙

### 도구(@tool) 구현 규칙

명세서 도구 정의 섹션의 각 도구를 구현할 때:
1. 함수명은 명세서 시그니처의 함수명과 **정확히 동일**하게 사용
2. 파라미터명, 타입, 기본값은 명세서 시그니처를 그대로 따름
3. docstring에 명세서의 **Purpose**와 **When to use**를 포함
4. `strands-sdk-guide` skill의 @tool 데코레이터 사용법을 따름
5. `USE_MOCK` 분기로 Mock 함수 호출 또는 `NotImplementedError` 처리

### Agents as Tools 패턴 구현 규칙

명세서 에이전트 컴포넌트 섹션에서 오케스트레이터의 도구 목록에 `invoke_{agent}` 형태의 도구가 있으면:
1. `strands-sdk-guide` skill에서 Agents as Tools 패턴을 확인
2. 해당 서브에이전트 인스턴스를 생성하고 @tool로 래핑
3. 오케스트레이터에 전달할 파라미터를 자연어 프롬프트로 구성하여 서브에이전트에 전달
4. 서브에이전트 응답을 파싱하여 구조화된 dict 반환

### 상태 관리 구현 규칙

명세서 상태 관리 섹션의 계층 구조를 Pydantic 모델로 정의:
- **Shared State**: 모든 에이전트가 읽을 수 있는 파이프라인 공유 데이터
- **Session State**: 단일 실행 세션 동안만 유지되는 중간 데이터
- **Persistent State**: 세션 종료 후에도 보존되는 이력/감사 데이터 (SQLite 또는 JSON)

### HiL 라우팅 규칙 구현

명세서에 정의된 Human-in-the-Loop 라우팅 조건을 **LLM이 아닌 Python 조건문**으로 구현:
- 신뢰도 임계값, 이력 건수 등의 판정은 결정론적 규칙으로
- 오케스트레이터 에이전트의 system_prompt에 이미 규칙이 명시되어 있으므로
  Python 레벨에서는 해당 규칙을 별도 함수로도 구현하여 중복 보호 계층 확보

### 오류 처리 구현 규칙

명세서에 정의된 오류 처리/Fallback 시나리오를 구현 (없으면 생략):
- 재시도 횟수, 백오프 간격, 타임아웃 등 수치는 `src/config.py`에 상수로 정의하고 참조
- Fallback 동작도 명세서에 기술된 동작 그대로 구현
- 모든 오류는 로그 기록 후 명세서의 최종 Fallback 경로로 진행

---

## 6단계: 구현 순서

명세서에 정의된 **모든 Phase의 전체 기능**을 구현하세요.
아래 순서로 구현하세요:

1. `pyproject.toml` — 의존성 설정 (Python 3.12, strands-agents, pydantic, boto3 등)
2. `.env.example` — 필요한 환경변수 목록
3. `src/config.py` — USE_MOCK 플래그, Bedrock 설정, 환경변수 로딩
4. `src/models/` — Pydantic 모델 전체 정의 (state.py 포함)
5. `src/prompts/` — 명세서 원본 프롬프트를 에이전트별 파일로 저장
6. `src/mock/mock_data.py` — 명세서 출력 형식 정의 및 예시 기반 Mock 상수 정의
7. `src/mock/mock_{domain}.py` — 도메인별 Mock 함수 구현
8. `src/tools/` — 명세서의 **모든** 도구 구현 (Mock 분기 포함)
9. `src/agents/` — 명세서의 **모든** 에이전트 구현 (SDK skill 참조)
10. `src/state/` — 상태 관리 구현
11. `src/main.py` — 진입점 구현
12. `tests/fixtures/` — 명세서 Example/Edge Case JSON 저장
13. `tests/test_e2e/test_normal_flow.py` — E2E 정상 흐름 테스트
14. `tests/test_e2e/test_edge_cases.py` — 명세서 Edge Case 전체 테스트

---

## 7단계: 검증 기준

구현 완료 후 아래 기준을 모두 충족하는지 확인하세요:

| 검증 항목 | 확인 방법 |
|-----------|-----------|
| 명세서의 시스템 프롬프트가 수정 없이 저장되었는가 | 명세서 원본과 `src/prompts/` 파일 diff 비교 |
| 명세서의 모든 도구가 동일한 시그니처로 구현되었는가 | 각 도구 파라미터명·타입 비교 |
| 명세서의 모든 에이전트가 구현되었는가 | `src/agents/` 디렉토리 파일 목록 확인 |
| `USE_MOCK=true`로 E2E 파이프라인이 오류 없이 완주하는가 | `tests/test_e2e/test_normal_flow.py` 실행 |
| Edge Case 입력에 대해 명세서 기술대로 처리되는가 | `tests/test_e2e/test_edge_cases.py` 실행 |
| 명세서에 정의된 모든 기능이 빠짐없이 구현되었는가 | 명세서 전체 섹션과 코드 대조 확인 |

---

## 8단계: 실행 방법 (README에 포함)

최종적으로 아래와 같이 실행 가능해야 합니다:

```bash
# 환경 설정
cp .env.example .env
# .env에 AWS_REGION, AWS_PROFILE 또는 AWS_ACCESS_KEY_ID 등 설정

# Python 3.12 가상환경 생성 및 의존성 설치
uv venv --python 3.12
source .venv/bin/activate
uv pip install -e .

# Mock 모드로 실행
USE_MOCK=true python -m src.main

# 테스트
pytest tests/ -v
```

---

## 참조 명세서

위에 지정한 명세서 파일을 구현 전에 전체를 읽으세요.
의문이 생길 때마다 명세서를 재참조하세요. 명세서에 없는 내용을 임의로 추가하지 마세요.