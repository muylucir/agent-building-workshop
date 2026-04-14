# Streamlit + Strands 패턴 상세 레시피

## 공통: app.py 기본 구조

모든 패턴에 공통으로 적용하는 app.py 뼈대.

```python
import streamlit as st
import traceback

st.set_page_config(page_title="에이전트 이름", layout="wide")
st.title("🤖 에이전트 이름")

# 세션 상태 초기화
if "pipeline_result" not in st.session_state:
    st.session_state.pipeline_result = None
    st.session_state.is_running = False

# 사이드바: 입력 + 실행
with st.sidebar:
    st.header("📝 입력")
    # ... 입력 폼 ...
    col1, col2 = st.columns(2)
    with col1:
        run_btn = st.button("▶️ 실행", disabled=st.session_state.is_running, use_container_width=True)
    with col2:
        if st.button("🔄 초기화", use_container_width=True):
            for key in list(st.session_state.keys()):
                del st.session_state[key]
            st.rerun()
```

---

## 1. Simple Wrapper (싱글 에이전트)

에이전트 1개를 감싸는 가장 단순한 패턴.

```python
from agents.some_agent import SomeAgent

if run_btn:
    st.session_state.is_running = True
    try:
        with st.status("🔄 에이전트 실행 중...", expanded=True) as status:
            st.write("입력 데이터 준비...")
            agent = SomeAgent()
            st.write("에이전트 추론 중...")
            result = agent.run(input_data)
            st.session_state.pipeline_result = result
            status.update(label="✅ 완료", state="complete")
    except Exception as e:
        st.error(f"❌ 오류: {e}")
        with st.expander("상세 정보"):
            st.code(traceback.format_exc())
    finally:
        st.session_state.is_running = False

# 결과 표시
if st.session_state.pipeline_result:
    st.subheader("📊 결과")
    st.json(st.session_state.pipeline_result)
```

---

## 2. Stage Progress (멀티 에이전트 순차)

여러 에이전트가 순서대로 실행될 때 단계별 진행 상황을 표시.

```python
from agents.agent_a import AgentA
from agents.agent_b import AgentB

STAGES = [
    ("1단계: 분석", AgentA),
    ("2단계: 생성", AgentB),
]

if run_btn:
    st.session_state.is_running = True
    results = {}
    try:
        with st.status("🔄 파이프라인 실행 중...", expanded=True) as status:
            current_input = input_data
            for i, (label, agent_cls) in enumerate(STAGES):
                st.write(f"{label}...")
                agent = agent_cls()
                result = agent.run(current_input)
                results[f"stage_{i+1}"] = result
                current_input = result  # 다음 단계 입력
            st.session_state.pipeline_result = results
            status.update(label="✅ 전체 완료", state="complete")
    except Exception as e:
        st.error(f"❌ {label} 실행 중 오류: {e}")
        with st.expander("상세 정보"):
            st.code(traceback.format_exc())
    finally:
        st.session_state.is_running = False

# 결과: 단계별 탭
if st.session_state.pipeline_result:
    tabs = st.tabs([name for name, _ in STAGES])
    for tab, (name, _) in zip(tabs, STAGES):
        with tab:
            st.json(st.session_state.pipeline_result.get(f"stage_{STAGES.index((name, _))+1}"))
```

---

## 3. Parallel Status (멀티 에이전트 병렬)

병렬 실행되는 에이전트들의 상태를 동시에 표시.

```python
import concurrent.futures

PARALLEL_AGENTS = {
    "분석 A": AgentA,
    "분석 B": AgentB,
}

if run_btn:
    st.session_state.is_running = True
    cols = st.columns(len(PARALLEL_AGENTS))
    results = {}

    def run_agent(name, agent_cls, data):
        agent = agent_cls()
        return name, agent.run(data)

    try:
        with st.status("🔄 병렬 실행 중...", expanded=True) as status:
            with concurrent.futures.ThreadPoolExecutor() as executor:
                futures = {
                    executor.submit(run_agent, name, cls, input_data): name
                    for name, cls in PARALLEL_AGENTS.items()
                }
                for future in concurrent.futures.as_completed(futures):
                    name, result = future.result()
                    results[name] = result
                    st.write(f"✅ {name} 완료")
            st.session_state.pipeline_result = results
            status.update(label="✅ 전체 완료", state="complete")
    except Exception as e:
        st.error(f"❌ 오류: {e}")
    finally:
        st.session_state.is_running = False
```

---

## 4. Chat Interface (대화형 에이전트)

사용자와 에이전트가 대화하는 인터페이스.

```python
from strands import Agent

if "messages" not in st.session_state:
    st.session_state.messages = []
    st.session_state.agent = None

def get_agent():
    if st.session_state.agent is None:
        st.session_state.agent = Agent(
            system_prompt=SYSTEM_PROMPT,
            model=model,
            tools=[...],
        )
    return st.session_state.agent

# 대화 이력 표시
for msg in st.session_state.messages:
    with st.chat_message(msg["role"]):
        st.markdown(msg["content"])

# 사용자 입력
if prompt := st.chat_input("메시지를 입력하세요"):
    st.session_state.messages.append({"role": "user", "content": prompt})
    with st.chat_message("user"):
        st.markdown(prompt)

    with st.chat_message("assistant"):
        with st.spinner("생각 중..."):
            agent = get_agent()
            response = agent(prompt)
            content = str(response)
        st.markdown(content)
    st.session_state.messages.append({"role": "assistant", "content": content})
```

---

## 5. Streaming Display (스트리밍 출력)

Strands Agent의 스트리밍 콜백을 Streamlit에 연결.

```python
from strands import Agent
from strands.models import BedrockModel

def create_streaming_agent(placeholder):
    """Streamlit placeholder에 스트리밍 출력을 연결한 Agent 생성"""
    accumulated = []

    def stream_callback(event):
        if "data" in event:
            accumulated.append(event["data"])
            placeholder.markdown("".join(accumulated))

    model = BedrockModel(
        model_id="global.anthropic.claude-sonnet-4-6",
        max_tokens=8192,
        streaming=True,
        callback_handler=stream_callback,
    )
    return Agent(system_prompt=SYSTEM_PROMPT, model=model, tools=[...])

if run_btn:
    output_area = st.empty()
    agent = create_streaming_agent(output_area)
    result = agent(user_input)
```

> 스트리밍은 선택 사항이다. 기본 패턴(Simple Wrapper, Stage Progress)으로 충분한 경우 스트리밍을 추가하지 않는다.

---

## 입력 폼 자동 생성 패턴

프로젝트의 schemas/models.py 또는 main.py 기본 입력을 분석하여 폼을 생성한다.

```python
import json

# main.py의 기본 입력을 기본값으로 사용
DEFAULT_INPUT = {
    "query": "예시 질문",
    "options": {"max_results": 5}
}

with st.sidebar:
    st.header("📝 입력")
    input_json = st.text_area(
        "입력 데이터 (JSON)",
        value=json.dumps(DEFAULT_INPUT, ensure_ascii=False, indent=2),
        height=200,
    )
    try:
        input_data = json.loads(input_json)
    except json.JSONDecodeError:
        st.error("올바른 JSON 형식이 아닙니다")
        input_data = None
```

개별 필드가 명확한 경우 (Pydantic 모델 기반):

```python
with st.sidebar:
    st.header("📝 입력")
    name = st.text_input("이름", value="홍길동")
    category = st.selectbox("분류", ["A", "B", "C"])
    count = st.number_input("수량", min_value=1, value=10)
    input_data = {"name": name, "category": category, "count": count}
```

---

## 6. Agent Observability (에이전트 동작 가시화)

Strands SDK의 HookProvider를 사용하여 에이전트 내부 이벤트를 Streamlit UI에 실시간으로 표시한다.

### StreamlitHook 클래스

```python
import time
import streamlit as st
from strands.hooks import (
    HookProvider, HookRegistry,
    BeforeInvocationEvent, AfterInvocationEvent,
    BeforeToolCallEvent, AfterToolCallEvent,
)

class StreamlitHook(HookProvider):
    """Strands Agent 이벤트를 Streamlit UI에 실시간 표시하는 Hook."""

    def __init__(self):
        self.tool_start_times: dict[str, float] = {}
        # session_state에 로그 리스트 초기화
        if "agent_logs" not in st.session_state:
            st.session_state.agent_logs = []

    def register_hooks(self, registry: HookRegistry) -> None:
        registry.add_callback(BeforeInvocationEvent, self.on_agent_start)
        registry.add_callback(AfterInvocationEvent, self.on_agent_end)
        registry.add_callback(BeforeToolCallEvent, self.on_tool_start)
        registry.add_callback(AfterToolCallEvent, self.on_tool_end)

    def on_agent_start(self, event: BeforeInvocationEvent):
        st.session_state.agent_logs.append({
            "type": "agent_start",
            "agent": getattr(event.agent, "name", "Agent"),
            "timestamp": time.time(),
        })

    def on_agent_end(self, event: AfterInvocationEvent):
        st.session_state.agent_logs.append({
            "type": "agent_end",
            "agent": getattr(event.agent, "name", "Agent"),
            "timestamp": time.time(),
        })

    def on_tool_start(self, event: BeforeToolCallEvent):
        tool_name = event.tool_use["name"]
        tool_input = event.tool_use.get("input", {})
        self.tool_start_times[tool_name] = time.time()
        st.session_state.agent_logs.append({
            "type": "tool_start",
            "tool": tool_name,
            "input": tool_input,
            "timestamp": time.time(),
        })

    def on_tool_end(self, event: AfterToolCallEvent):
        tool_name = event.tool_use["name"]
        tool_output = event.tool_result
        elapsed = time.time() - self.tool_start_times.pop(tool_name, time.time())
        st.session_state.agent_logs.append({
            "type": "tool_end",
            "tool": tool_name,
            "output": tool_output,
            "elapsed_ms": round(elapsed * 1000),
            "timestamp": time.time(),
        })
```

### Hook을 에이전트에 주입

기존 에이전트 코드를 수정하지 않고, app.py에서 에이전트 생성 후 hook을 추가한다.

```python
from src.agents.orchestrator import OrchestratorAgent

hook = StreamlitHook()

# 방법 1: 에이전트 클래스가 Agent 인스턴스를 속성으로 노출하는 경우
agent = OrchestratorAgent()
agent.agent.hooks.add_hook_provider(hook)

# 방법 2: Agent를 직접 생성하는 경우
from strands import Agent
agent = Agent(system_prompt=PROMPT, model=model, tools=tools, hooks=[hook])
```

### 로그를 UI에 렌더링

```python
def render_agent_logs():
    """session_state에 쌓인 로그를 Streamlit UI로 렌더링"""
    for log in st.session_state.get("agent_logs", []):
        if log["type"] == "agent_start":
            st.info(f"🤖 **{log['agent']}** 시작")
        elif log["type"] == "agent_end":
            st.success(f"✅ **{log['agent']}** 완료")
        elif log["type"] == "tool_start":
            with st.expander(f"🔧 도구: {log['tool']}", expanded=False):
                st.caption("입력")
                st.json(log["input"])
                # tool_end 로그 찾아서 출력 표시
                matching_end = next(
                    (l for l in st.session_state.agent_logs
                     if l["type"] == "tool_end" and l["tool"] == log["tool"]
                     and l["timestamp"] > log["timestamp"]),
                    None
                )
                if matching_end:
                    st.caption("출력")
                    st.json(matching_end["output"])
                    st.caption(f"⏱️ {matching_end['elapsed_ms']}ms")
```

### 전체 app.py 통합 예시

```python
if run_btn:
    st.session_state.is_running = True
    st.session_state.agent_logs = []  # 로그 초기화
    hook = StreamlitHook()

    try:
        with st.status("🔄 파이프라인 실행 중...", expanded=True) as status:
            agent = OrchestratorAgent()
            agent.agent.hooks.add_hook_provider(hook)
            result = agent.run(input_data)
            st.session_state.pipeline_result = result
            status.update(label="✅ 완료", state="complete")
    except Exception as e:
        st.error(f"❌ 오류: {e}")
    finally:
        st.session_state.is_running = False

    # 에이전트 동작 로그 표시
    st.subheader("📋 에이전트 동작 로그")
    render_agent_logs()

    # 최종 결과
    if st.session_state.pipeline_result:
        st.subheader("📊 최종 결과")
        st.json(st.session_state.pipeline_result)
```
