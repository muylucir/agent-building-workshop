# PATH Agent Builder Workshop — Claude Code Harness

이 디렉토리는 PATH 명세서(`.md`)를 Strands Agents SDK 기반 Python 코드로 변환하기 위한 Claude Code 워크샵 환경이다.

## 사용법

1. PATH 명세서 파일(`.md`)을 이 디렉토리 또는 하위 경로에 둔다.
2. Claude Code 세션에서 다음 중 하나를 실행:
   - **풀 파이프라인**: `/build-from-spec <명세서경로>` — agent-builder → mock-builder → evaluator → fixer → 재평가까지 자동 진행
   - **단계별 실행**: 아래 subagent를 직접 호출
     - `path-agent-builder` — 전체 코드 구현
     - `path-mock-builder` — mock 품질 개선
     - `path-spec-evaluator` — 명세서 충실도 평가
     - `path-spec-fixer` — 평가 리포트 기반 수정

## 프로젝트 가이드 문서 (`docs/`)

모든 작업은 아래 문서의 규칙을 기반으로 한다. 작업 시작 전 관련 문서를 읽어라.

@docs/product.md
@docs/structure.md
@docs/tech.md
@docs/spec-to-code.md
@docs/pipeline.md
@docs/mock.md
@docs/evaluation.md

## Skills (`.claude/skills/`)

- `strands-sdk-guide` — Strands SDK 패턴, @tool, 멀티에이전트, 세션, hooks
- `mock-patterns` — mock 구현 테크닉

## 핵심 원칙

- 명세서의 시스템 프롬프트는 **한 글자도 수정 없이** 그대로 사용
- 외부 API는 모두 현실적 mock으로 구현 (`mocks/`)
- 명세서의 Next Steps는 참고용 — **모든 에이전트·도구·상태·에러처리를 전부 구현**
- Python 3.12 + Strands Agents SDK만 사용 (LangChain/CrewAI 금지)
