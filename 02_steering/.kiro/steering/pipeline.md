# 구현 파이프라인

PATH 명세서를 코드로 변환하는 전체 워크플로우. 각 단계는 전용 서브에이전트가 담당한다.

## 파이프라인

```
명세서(.md) → ① agent-builder → ② mock-builder → ③ evaluator → ④ fixer → ⑤ 재평가 (95%+) → ⑥ streamlit-builder
```

### 1. `path-agent-builder` — 전체 코드 구현

- 명세서를 읽고 프로젝트 디렉토리를 생성
- 에이전트, 프롬프트, 도구, 상태관리, 에러처리, mock을 **모두 포함**하여 구현
- mock은 E2E 실행이 가능한 초기 버전으로 생성 (품질 개선은 다음 단계)

### 2. `path-mock-builder` — mock 품질 개선/보강

- 구현된 코드의 `mocks/`, `mocks/data/`, `tools/`를 읽고 품질 진단
- 개선 항목:
  - mock 데이터 5건 이상 확보
  - 입력 기반 분기 로직 (정적 반환 → 키워드 매칭/룩업 등)
  - 에러 시뮬레이션 (`__TIMEOUT__`, `__ERROR__`, `__NOT_FOUND__`)
  - 명세서 `<output_format>` JSON 구조 일치 검증

### 3. `path-spec-evaluator` — 명세서 충실도 평가

- 명세서와 구현 코드를 비교하여 7개 영역 평가 (evaluation.md 기준)
- 영역: 에이전트 완전성, 프롬프트 충실도, 도구 완전성, 상태관리, 에러처리, Mock 품질, 아키텍처 일치도
- 종합 점수 및 등급 산출 (S/A/B/C/D)
- `evaluation-report-v{N}.md` 파일로 저장

### 4. `path-spec-fixer` — Critical/Warning 이슈 수정

- 평가 리포트의 🔴 Critical (누락) → 🟡 Warning (불일치) 순으로 수정
- 명세서를 원본 기준으로 사용
- `fix-report-v{N}.md` 파일로 변경 사항 기록

### 5. 재평가 반복

- 수정 후 `path-spec-evaluator`를 다시 실행
- 종합 점수 **95% 이상 (S등급)** 도달까지 3→4→5 반복

### 6. `path-streamlit-builder` — Streamlit UI 생성

- 품질 검증이 완료된 프로젝트에 `app.py`를 추가
- 기존 코드(agents/, tools/, mocks/, main.py 등)를 **수정하지 않고** import하여 UI로 감쌈
- 비개발자가 `streamlit run app.py`로 브라우저에서 에이전트를 실행·확인 가능
- streamlit.md steering + streamlit-patterns skill 기준으로 생성
