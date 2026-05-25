# PATH Agent Builder — Claude Code (원샷 프롬프트 버전)

가장 가벼운 형태의 워크샵 환경. Custom subagent나 steering 없이, **하나의 프롬프트**와 **두 개의 skill**만으로 명세서를 코드로 변환한다.

## 디렉토리

```
03_claude/
├── .claude/
│   └── skills/
│       └── strands-sdk-guide/    # Strands SDK 패턴
└── prompt/
    └── agent-building-prompt.md  # 원샷 프롬프트 (사용자가 복붙)
```

## 사용법

1. 이 디렉토리에서 `claude` 실행
2. 명세서 파일을 `prompt/` 또는 임의의 위치에 둔다
3. `prompt/agent-building-prompt.md` 내용을 복붙하되, `명세서 파일` 경로를 본인 명세서 경로로 수정
4. Skill은 `from strands import` 같은 키워드를 만나면 자동으로 트리거됨

## 04_claude_harness와의 차이

| 항목 | 03_claude | 04_claude_harness |
|------|-----------|-------------------|
| Subagents | ❌ | ✅ 4개 (builder, mock-builder, evaluator, fixer) |
| 가이드 문서 (docs/) | ❌ | ✅ 7개 (product, structure, tech, spec-to-code, pipeline, mock, evaluation) — CLAUDE.md에서 @import |
| Slash command | ❌ | ✅ `/build-from-spec` 풀 파이프라인 |
| Skills | ✅ 1개 (strands-sdk-guide) | ✅ 2개 (strands-sdk-guide, mock-patterns) |
| 진입 방식 | 사용자가 프롬프트 복붙 | subagent 직접 호출 또는 슬래시 커맨드 |
| 평가/재수정 루프 | ❌ | ✅ S등급 95%까지 자동 반복 |
