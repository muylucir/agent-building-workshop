---
name: streamlit-patterns
description: |
  Streamlit + Strands Agents SDK 통합 패턴 레시피. 에이전트 파이프라인을 Streamlit UI로 감싸는 구현 테크닉을 제공한다.
  TRIGGER: Streamlit app.py 생성, 에이전트 실행 결과를 브라우저에 표시, 스트리밍 출력 연동, 입력 폼 자동 생성, 세션 상태 관리 시 사용.
---

# Streamlit + Strands 통합 패턴

에이전트 프로젝트의 기존 파이프라인을 Streamlit UI로 감쌀 때 사용하는 패턴 가이드.

## 품질 기준

| 항목 | 불량 | 양호 |
|------|------|------|
| 기존 코드 영향 | 기존 파일 수정 | app.py만 추가, 기존 코드 무수정 |
| 입력 폼 | 하드코딩된 필드 | 스키마 기반 자동 생성 |
| 실행 피드백 | 빈 화면에서 대기 | 단계별 진행 상태 실시간 표시 |
| 에러 처리 | 앱 크래시 | st.error()로 안내 + 상세 정보 접기 |
| 결과 표시 | raw JSON 덤프 | 구조화된 마크다운/테이블/JSON 뷰 |

## 패턴 선택 기준

| 프로젝트 특성 | 판별 기준 | 패턴 |
|-------------|----------|------|
| **싱글 에이전트** | agents/ 에 1개 파일, 도구 직접 호출 | Simple Wrapper |
| **멀티 에이전트 (순차)** | 여러 에이전트가 순서대로 실행 | Stage Progress |
| **멀티 에이전트 (병렬)** | asyncio.gather 또는 concurrent.futures 사용 | Parallel Status |
| **대화형 에이전트** | 사용자 입력 → 응답 → 추가 입력 루프 | Chat Interface |
| **스트리밍 필요** | 긴 추론 과정을 실시간으로 보여줘야 함 | Streaming Display |

## 상세 레시피

각 패턴의 구현 코드, app.py 구조, 세션 관리는 → [references/patterns.md](references/patterns.md) 참조
