---
description: PATH 명세서로부터 풀 파이프라인을 실행 (구현 → mock 보강 → 평가 → 수정 → 재평가, S등급 95%+ 도달까지)
argument-hint: <명세서 .md 파일 경로>
---

PATH 명세서 `$ARGUMENTS`로부터 다음 5단계 파이프라인을 순차 실행한다. 각 단계는 전용 subagent에 위임한다.

## 실행 절차

1. **path-agent-builder 호출** — 명세서를 전체 구현. 프로젝트 디렉토리가 생성되고 코드가 작성된다.
2. **path-mock-builder 호출** — 1단계가 끝난 후, 같은 프로젝트 디렉토리의 mock 품질을 개선한다.
3. **path-spec-evaluator 호출** — 명세서와 구현 디렉토리를 비교 평가하여 `evaluation-report-v1.md`를 생성한다.
4. **path-spec-fixer 호출** — 평가 리포트의 🔴 Critical, 🟡 Warning 이슈를 수정하고 `fix-report-v1.md`를 생성한다.
5. **path-spec-evaluator 재호출** — 수정 후 재평가. 종합 점수가 **95% 이상 (S등급)** 이 되면 종료. 그렇지 않으면 4→5를 반복한다.

## 규칙

- 각 단계 완료 후, 다음 단계로 넘어가기 전에 결과를 한 줄로 요약 보고한다.
- 5단계 반복 횟수가 3회를 초과하면 사용자에게 진행 여부를 묻는다.
- subagent 호출은 Task 도구를 사용하며, 한 번에 하나씩 (의존성이 있으므로 순차).
- `docs/pipeline.md`, `docs/evaluation.md`, `docs/spec-to-code.md`의 규칙을 따른다.
