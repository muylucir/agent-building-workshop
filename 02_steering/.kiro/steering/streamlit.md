# Streamlit UI 생성 규칙

구현 완료된 에이전트 프로젝트에 Streamlit UI를 추가할 때 따라야 할 규칙.

## 원칙

1. **기존 코드 무수정** — agents/, tools/, mocks/, prompts/, schemas/, config.py, main.py를 절대 수정하지 않는다
2. **추가만 허용** — `app.py` 신규 생성, `requirements.txt`에 streamlit 추가, `README.md`에 실행법 추가
3. **한국어 UI** — 모든 라벨, 안내 문구, 에러 메시지는 한국어
4. **비개발자 대상** — 터미널 지식 없이 브라우저에서 입력 → 실행 → 결과 확인이 가능해야 함

## 파일 구조

```
project-root/
├── app.py                     # Streamlit 진입점 (신규)
├── main.py                    # 기존 CLI 진입점 (수정 금지)
├── agents/ ...                # 기존 코드 (수정 금지)
└── requirements.txt           # streamlit>=1.45 추가
```

## 레이아웃 구조

3영역 레이아웃을 기본으로 한다.

| 영역 | Streamlit 구현 | 내용 |
|------|---------------|------|
| 사이드바 | `st.sidebar` | 입력 폼 + 실행 버튼 + 리셋 버튼 |
| 메인 상단 | `st.container` | 실행 진행 상황 (단계별 상태) |
| 메인 하단 | `st.container` | 최종 결과 출력 |

## 입력 폼 생성 규칙

프로젝트의 입력 스키마(schemas/models.py 또는 main.py의 기본 입력)를 분석하여 자동으로 폼을 생성한다.

| 입력 타입 | Streamlit 컴포넌트 |
|----------|-------------------|
| `str` (짧은 텍스트) | `st.text_input()` |
| `str` (긴 텍스트) | `st.text_area()` |
| `int` | `st.number_input(step=1)` |
| `float` | `st.number_input(step=0.01)` |
| `bool` | `st.checkbox()` |
| `Literal[...]` / enum | `st.selectbox()` |
| `list[str]` | `st.text_area()` + 줄바꿈 파싱 |
| `dict` / 복합 객체 | `st.text_area()` + JSON 파싱 |

## 실행 흐름 표시

### 단계별 진행 상황

```python
with st.status("🔄 파이프라인 실행 중...", expanded=True) as status:
    st.write("1단계: 입력 검증...")
    # ... 실행 ...
    st.write("2단계: 에이전트 실행...")
    # ... 실행 ...
    status.update(label="✅ 완료", state="complete")
```

### 에이전트 추론 과정

```python
with st.expander("🤖 에이전트 추론 과정", expanded=False):
    st.markdown(reasoning_text)
```

### 도구 호출 로그

```python
with st.expander(f"🔧 도구 호출: {tool_name}", expanded=False):
    st.json(tool_input)
    st.json(tool_output)
```

## 결과 출력 규칙

| 결과 타입 | Streamlit 컴포넌트 |
|----------|-------------------|
| 텍스트 (마크다운) | `st.markdown()` |
| JSON 구조 | `st.json()` |
| 테이블 데이터 | `st.dataframe()` |
| 성공/실패 상태 | `st.success()` / `st.error()` |
| 경고/참고 | `st.warning()` / `st.info()` |

## 세션 상태 관리

```python
# 초기화
if "pipeline_result" not in st.session_state:
    st.session_state.pipeline_result = None
    st.session_state.is_running = False

# 리셋
if st.sidebar.button("🔄 초기화"):
    for key in list(st.session_state.keys()):
        del st.session_state[key]
    st.rerun()
```

## Strands Agent 연동 패턴

기존 파이프라인을 import하여 호출한다. 파이프라인 코드를 복사하거나 재구현하지 않는다.

### 싱글 에이전트

```python
from agents.<agent_name> import SomeAgent

agent = SomeAgent()
result = agent.run(input_data)
```

### 멀티 에이전트 / 파이프라인

```python
# main.py의 run_pipeline 또는 동등한 함수를 import
from main import run_pipeline

result = run_pipeline(input_data)
```

### 스트리밍 출력 (선택)

Strands Agent의 스트리밍을 Streamlit에 연결하려면 streamlit-patterns skill의 streaming 레시피를 참조한다.

## 에러 처리

```python
try:
    result = run_pipeline(input_data)
    st.success("✅ 실행 완료")
except Exception as e:
    st.error(f"❌ 실행 중 오류가 발생했습니다: {e}")
    with st.expander("상세 오류 정보"):
        st.code(traceback.format_exc())
```

## 에이전트 동작 가시화 (Observability)

비개발자가 "AI가 지금 무엇을 하고 있는지"를 실시간으로 이해할 수 있어야 한다. 단순히 "실행 중..."만 보여주는 것은 금지.

### 필수 표시 항목

| 항목 | 표시 방법 | 설명 |
|------|----------|------|
| 현재 활성 에이전트 | `st.status()` 라벨 | 어떤 에이전트가 지금 동작 중인지 이름과 역할 표시 |
| 도구 호출 | `st.expander()` + 입출력 JSON | 어떤 도구를 왜 호출했고, 무엇을 받았는지 |
| 에이전트 판단 | `st.info()` | 에이전트가 내린 핵심 결정 (분류 결과, 라우팅 선택 등) |
| 단계 간 데이터 흐름 | `st.expander()` | 이전 에이전트 출력 → 다음 에이전트 입력으로 전달된 데이터 |
| 최종 결과 | `st.success()` + 구조화 출력 | 파이프라인 최종 산출물 |

### Strands Hooks 기반 실시간 로깅

Strands SDK의 `HookProvider`를 사용하여 에이전트 내부 이벤트를 Streamlit UI에 실시간으로 표시한다. `app.py`에 `StreamlitHook` 클래스를 정의하고, 에이전트 생성 시 주입한다.

캡처해야 할 이벤트:

| Strands 이벤트 | UI 표시 |
|---------------|---------|
| `BeforeInvocationEvent` | `st.status()` — "🤖 {agent_name} 시작" |
| `AfterInvocationEvent` | `st.status()` — "✅ {agent_name} 완료" |
| `BeforeToolCallEvent` | `st.expander("🔧 도구: {tool_name}")` — 입력 파라미터 표시 |
| `AfterToolCallEvent` | 같은 expander에 반환값 추가 |

구현 패턴은 streamlit-patterns skill의 **Agent Observability** 레시피를 참조한다.

### 에이전트별 구역 분리

멀티 에이전트 파이프라인에서는 각 에이전트의 동작을 시각적으로 구분한다:

```python
for agent_name, agent_fn in pipeline_stages:
    with st.container(border=True):
        st.subheader(f"🤖 {agent_name}")
        with st.status(f"{agent_name} 실행 중...", expanded=True) as status:
            # hook이 이 컨테이너 안에 도구 호출 로그를 추가
            result = agent_fn(current_input)
            status.update(label=f"✅ {agent_name} 완료", state="complete")
        with st.expander(f"📤 {agent_name} 출력 데이터"):
            st.json(result)
```

### 도구 호출 표시 형식

도구 호출은 입력과 출력을 명확히 구분하여 표시한다:

```python
with st.expander(f"🔧 도구: {tool_name}", expanded=False):
    st.caption("입력")
    st.json(tool_input)
    st.caption("출력")
    st.json(tool_output)
    st.caption(f"⏱️ {elapsed_ms}ms")
```

## 필수 규칙

1. `app.py`는 프로젝트 루트에 생성한다
2. 기존 코드의 import 경로를 변경하지 않는다 — `app.py`에서 기존 모듈을 그대로 import
3. `st.set_page_config()`를 파일 최상단에 호출한다 (layout="wide")
4. 실행 중 상태를 `st.session_state.is_running`으로 관리하여 중복 실행을 방지한다
5. 모든 긴 작업은 `st.spinner()` 또는 `st.status()`로 감싸서 진행 상태를 표시한다
6. JSON 입력 필드에는 기본값으로 main.py의 예시 입력을 미리 채워넣는다
