# Mock 패턴 상세 레시피

## 공통: 에러 시뮬레이션

모든 패턴에 공통으로 적용한다. mock 함수 진입부에서 특수 입력값을 체크한다.

```python
def _check_error_simulation(value: str):
    """Check for error simulation triggers. Call at the top of every mock function."""
    if not isinstance(value, str):
        return
    v = value.upper()
    if v == "__TIMEOUT__":
        raise TimeoutError("Simulated timeout")
    if v == "__ERROR__":
        raise RuntimeError("Simulated internal error")
    if v == "__NOT_FOUND__":
        return None  # caller handles empty result
```

---

## 1. Keyword Matching (검색/탐색)

자연어 쿼리를 받아 데이터의 키워드와 매칭하여 유사도 순으로 반환.

### 데이터 구조 (`mocks/data/`)

```json
[
  {
    "id": "REC-001",
    "title": "...",
    "description": "...",
    "keywords": ["keyword1", "keyword2", "keyword3"],
    "metadata": {}
  }
]
```

`keywords` 필드가 핵심. 명세서 `<examples>`에서 도메인 용어를 추출하여 채운다.

### mock 함수

```python
import json
from pathlib import Path

_DATA_DIR = Path(__file__).parent / "data"
_RECORDS = json.loads((_DATA_DIR / "<domain>_records.json").read_text())

def mock_search(query: str, top_k: int = 10) -> list:
    _check_error_simulation(query)
    query_tokens = set(query.lower().split())
    scored = []
    for rec in _RECORDS:
        hits = sum(1 for kw in rec["keywords"] if kw in query_tokens or any(kw in t for t in query_tokens))
        if hits > 0:
            scored.append({**rec, "score": round(hits / len(rec["keywords"]), 2)})
    return sorted(scored, key=lambda x: x["score"], reverse=True)[:top_k]
```

### 개선 체크리스트

- [ ] 키워드가 도메인에 맞는 실제 용어인가 (더미 아닌지)
- [ ] 쿼리 토큰화가 부분 매칭을 지원하는가
- [ ] top_k 파라미터가 동작하는가
- [ ] 매칭 결과가 0건일 때 빈 리스트를 반환하는가

---

## 2. Lookup Table (코드/분류 조회)

정확한 ID나 코드로 단건 조회. dict 기반 O(1) 룩업.

### 데이터 구조

```json
{
  "CODE-001": {
    "code": "CODE-001",
    "name": "...",
    "category": "...",
    "details": {}
  },
  "CODE-002": { ... }
}
```

### mock 함수

```python
_LOOKUP = json.loads((_DATA_DIR / "<domain>_lookup.json").read_text())

def mock_get_by_code(code: str) -> dict | None:
    _check_error_simulation(code)
    return _LOOKUP.get(code.upper())
```

### 개선 체크리스트

- [ ] 대소문자 정규화가 되어 있는가
- [ ] 존재하지 않는 코드에 None을 반환하는가
- [ ] 반환 구조가 명세서 `<output_format>`과 일치하는가

---

## 3. Stateful CRUD (상태 변경)

인메모리 dict로 생성/조회/수정/삭제를 시뮬레이션.

### mock 함수

```python
_STORE: dict[str, dict] = {}

def mock_create(item: dict) -> dict:
    _check_error_simulation(item.get("id", ""))
    item_id = item.get("id") or f"AUTO-{len(_STORE)+1:04d}"
    _STORE[item_id] = {**item, "id": item_id, "status": "created"}
    return _STORE[item_id]

def mock_get(item_id: str) -> dict | None:
    _check_error_simulation(item_id)
    return _STORE.get(item_id)

def mock_update(item_id: str, updates: dict) -> dict | None:
    _check_error_simulation(item_id)
    if item_id not in _STORE:
        return None
    _STORE[item_id].update(updates)
    return _STORE[item_id]

def mock_delete(item_id: str) -> bool:
    _check_error_simulation(item_id)
    return _STORE.pop(item_id, None) is not None
```

### 개선 체크리스트

- [ ] 자동 ID 생성이 있는가
- [ ] 존재하지 않는 항목 수정/삭제 시 적절히 처리하는가
- [ ] 상태 변경 후 변경된 객체를 반환하는가

---

## 4. Deterministic Compute (계산/변환)

입력 수치를 받아 공식 기반으로 계산. 랜덤 요소 없이 동일 입력 → 동일 출력.

### mock 함수

```python
def mock_calculate(amount: float, rate: float, **params) -> dict:
    _check_error_simulation(str(amount))
    result = round(amount * rate, 2)
    return {
        "input_amount": amount,
        "rate": rate,
        "result": result,
        "currency": params.get("currency", "USD")
    }
```

### 개선 체크리스트

- [ ] 계산 공식이 도메인에 맞는가 (명세서 참조)
- [ ] 소수점 처리가 적절한가
- [ ] 경계값(0, 음수, 매우 큰 수)을 처리하는가

---

## 5. Fire-and-Confirm (외부 시스템 연동)

사이드이펙트(알림, 이메일, 웹훅)를 시뮬레이션. 항상 성공 응답을 반환하되 호출 기록을 남긴다.

### mock 함수

```python
_CALL_LOG: list[dict] = []

def mock_send_notification(recipient: str, message: str, channel: str = "email") -> dict:
    _check_error_simulation(recipient)
    record = {
        "recipient": recipient,
        "message": message,
        "channel": channel,
        "status": "sent",
        "timestamp": "2025-01-15T10:30:00Z"
    }
    _CALL_LOG.append(record)
    return record

def get_call_log() -> list[dict]:
    """For testing — inspect what was 'sent'."""
    return list(_CALL_LOG)
```

### 개선 체크리스트

- [ ] 호출 기록을 남기는가 (디버깅/검증용)
- [ ] 반환 구조에 status, timestamp가 포함되는가
- [ ] 채널/수단별 분기가 있는가 (명세서에 여러 채널이 정의된 경우)
