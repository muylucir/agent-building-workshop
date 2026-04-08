# Product Overview

## 목적

이 프로젝트는 **PATH(P.A.T.H Agent Designer)가 생성한 AI Agent 명세서**를 입력으로 받아, 실제 동작하는 AI Agent 코드를 구현하는 프로젝트다.

명세서는 마크다운 파일(`.md`)로 제공되며, 에이전트의 설계·프롬프트·도구·상태관리·에러처리가 모두 정의되어 있다. 개발자의 역할은 이 명세서를 코드로 충실히 변환하는 것이다.

**외부 API/데이터소스는 현실적인 mock으로 구현한다.** 명세서에 정의된 도구들이 호출하는 실제 외부 시스템은 존재하지 않으므로, 명세서의 예시 데이터를 기반으로 도메인에 맞는 현실적인 mock을 만들어 에이전트가 E2E로 동작하도록 한다.

## PATH 명세서 구조

명세서는 8개 섹션으로 구성된다:

| # | 섹션 | 내용 |
|---|------|------|
| 1 | **Executive Summary** | Problem, Solution(3계층 패턴), Feasibility 점수, Automation Level(Agentic AI / AI-Assisted Workflow), Recommendation |
| 2 | **Agent Design Pattern** | 3계층 택소노미(Layer1 Agent Pattern + Layer2 LLM Workflow + Layer3 Agentic Workflow), Agent Components 테이블, 에이전트별 상세 설계(책임·인지패턴·도구·판단로직), State Management, Error Handling |
| 3 | **Visual Design** | Agent Workflow(flowchart), Sequence Diagram, Architecture Overview — 모두 Mermaid 다이어그램 |
| 4 | **Agent Prompts** | 에이전트별 System Prompt(XML 구조) + Example User Prompt |
| 5 | **Tool Definitions** | Compact Signature 형식의 도구 정의 (Purpose, Signature, When to use) |
| 6 | **Problem Decomposition** | INPUT → PROCESS(단계별) → OUTPUT → Human-in-Loop 수준 |
| 7 | **Risks** | 주요 리스크 목록 |
| 8 | **Next Steps** | Phase별 구현 로드맵 |

## 두 가지 자동화 수준

명세서의 `Automation Level`에 따라 구현 패턴이 근본적으로 달라진다:

### Agentic AI (자율성 ≥6)
- 에이전트가 자율적으로 도구 선택, 판단, 반복 수행
- Layer3에 멀티 에이전트 패턴 존재 (Agents as Tools / Swarm / Graph / Workflow)
- 명세서에 Supervisor + Specialist 에이전트 구조가 정의됨

### AI-Assisted Workflow (자율성 ≤5)
- 워크플로우 엔진이 전체 흐름 제어, AI는 각 단계에서 보조
- Layer3는 "해당 없음" 또는 Conditional Pipeline
- 명세서에 Stage 1~N 순차 파이프라인이 정의됨
- 명세서 섹션 제목이 "Agent Components" 대신 "워크플로우 단계(Stages)"로 표기될 수 있음

## 코드 생성 대상 vs 참고용

| 명세서 섹션 | 코드 생성 | 참고용 |
|------------|:---------:|:------:|
| Agent Design Pattern (Components, State, Error) | ✅ | |
| Agent Prompts | ✅ | |
| Tool Definitions | ✅ | |
| Visual Design (Mermaid) | | ✅ |
| Problem Decomposition | | ✅ |
| Risks | | ✅ |
| Next Steps | | ✅ 참고 — 구현 범위 제한에 사용하지 않음 |
