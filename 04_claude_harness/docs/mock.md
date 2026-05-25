# Mock 구현 가이드

명세서의 Tool Definitions에 정의된 외부 API/데이터소스는 실제 시스템이 존재하지 않으므로, **현실적인 mock**으로 구현하여 에이전트가 E2E로 동작하도록 한다.

## 원칙

1. **단순 더미 데이터 금지** — `return "mock result"` 같은 구현은 허용하지 않음
2. **도메인에 맞는 현실적 데이터** — 명세서의 Example User Prompt와 출력 예시를 기반으로 실제와 유사한 데이터를 생성
3. **명세서 예시 활용** — Agent Prompts 섹션의 `<examples>` 블록에 있는 입출력 예시를 mock 데이터의 기준으로 사용
4. **변동성 포함** — 항상 같은 결과가 아니라, 입력에 따라 다른 결과를 반환하도록 구현
5. **에러 케이스 포함** — 명세서 Error Handling 테이블의 오류 시나리오를 mock에서 시뮬레이션할 수 있도록 구현

## 디렉토리 구조

```
mocks/
├── <domain>_mock.py           # 도메인별 mock 로직
├── ...
└── data/                      # mock용 정적 데이터 파일
    ├── <domain>_records.json
    └── ...
```

## 도구 함수와 mock 분리

도구 함수(`tools/`)는 mock 구현(`mocks/`)을 import하여 위임한다. 나중에 실제 API로 교체할 때 도구 함수의 import만 변경하면 된다.

도구 함수 (tools/):
```python
from strands import tool
from mocks.<domain>_mock import mock_<tool_name>

@tool
def <tool_name>(query: str, top_k: int = 10) -> list:
    """<명세서 Purpose 그대로 사용>

    Args:
        query: <Signature 파라미터 설명>
        top_k: 반환할 최대 후보 수
    """
    return mock_<tool_name>(query, top_k)
```

## mock 구현 패턴

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

## mock 데이터 파일

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

### 데이터 소스 우선순위

1. 명세서 `<examples>` 블록의 입출력 예시 (최우선)
2. 명세서 Tool Definitions의 반환 타입 구조
3. 명세서 State Management의 필드 설명
4. 도메인 지식 기반 합리적 생성

## 명세서 → mock 변환 규칙

명세서의 Compact Signature 형식:
```
- Purpose: 관세율표 DB를 RAG로 검색하여 후보 HS코드 목록을 반환
- Signature: tariff_schedule_retriever(item_description: str, top_k?: int = 10) -> list[{hs_code: str, description: str}]
- When to use: HS Code Classifier Agent가 분류 후보 코드를 탐색할 때
```

이 정보로부터:
1. `tools/<domain>_tools.py`에 `@tool` 함수 생성 — Purpose를 docstring으로, Signature를 함수 시그니처로
2. `mocks/<domain>_mock.py`에 mock 함수 생성 — 키워드 매칭/룩업 테이블 기반
3. `mocks/data/<domain>_records.json`에 명세서 `<examples>`에서 추출한 데이터 저장

## 필수 규칙

1. **명세서 `<output_format>`의 JSON 구조를 정확히 준수** — mock 반환값이 명세서 출력 스키마와 일치해야 함
2. **명세서 `<examples>` 블록의 입출력 데이터를 mock 데이터의 기준으로 사용** — 명세서 예시가 mock을 통과하면 에이전트가 정상 동작하는 것
3. **입력에 따라 결과가 달라지도록 구현** — 키워드 매칭, 조건 분기, 룩업 테이블 등 활용
4. **에러 시뮬레이션 지원** — 특정 입력값(예: `"__TIMEOUT__"`)으로 타임아웃/오류를 트리거할 수 있도록 구현
5. **mock 데이터는 최소 5건 이상** — 에이전트가 의미 있는 비교/선택을 할 수 있을 만큼의 데이터 확보

## Mock 품질 개선 서브에이전트 (`path-mock-builder`)

`path-agent-builder`가 구현한 코드의 mock을 명세서 기준으로 **개선/보강**하는 전용 에이전트.

### 실행 시점

`path-agent-builder`로 전체 코드 구현이 **완료된 후**에 실행한다. 이미 존재하는 `mocks/`, `mocks/data/`, `tools/`의 품질을 높이는 역할이다.

### 입력

- PATH 명세서 파일 경로 (`.md`)
- 구현 완료된 프로젝트 디렉토리 경로

### 개선 대상

| 대상 | 개선 내용 |
|------|----------|
| `mocks/data/*.json` | 데이터 건수 보강 (최소 5건), 명세서 `<examples>` 블록과 일치 여부 검증, 도메인 현실성 향상 |
| `mocks/*.py` | 단순 더미 반환 → 입력 기반 분기 로직으로 개선, 에러 시뮬레이션 추가 |
| `tools/*.py` | mock 반환값이 명세서 `<output_format>` JSON 구조와 일치하는지 검증 |

### 동작 순서

1. 명세서의 Tool Definitions, `<examples>`, `<output_format>`, Error Handling 섹션을 읽음
2. 구현된 `mocks/`, `tools/` 코드를 읽고 현재 품질을 진단
3. 진단 결과를 요약 출력 (도구별 mock 품질 점수, 누락 항목)
4. 개선 작업 수행:
   - mock 데이터 건수 부족 → 명세서 예시 + 도메인 지식으로 확장
   - 정적 반환 → 입력 기반 분기 로직으로 리팩터링
   - 에러 시뮬레이션 미지원 → 특수 입력값 처리 추가
   - 출력 스키마 불일치 → 명세서 `<output_format>` 기준으로 수정
5. 변경 사항 요약 리포트 출력