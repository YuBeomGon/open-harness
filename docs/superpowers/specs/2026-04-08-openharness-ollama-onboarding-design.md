# OpenHarness + Ollama Onboarding Package Design

Date: 2026-04-08
Status: Approved for planning
Audience: Claude Code/Codex를 조금 써봤고, Agent Harness는 얕게 아는 1인 사용자
Language: 한국어 기준, 핵심 기술 용어는 English 유지

## 1. Background

현재 저장소는 [README.md](../../../README.md)에 프로젝트 개요, feature, provider, architecture 설명이 많이 모여 있고, `docs/`에는 [SHOWCASE.md](../../SHOWCASE.md)만 있는 상태다.

이 구조는 처음 접하는 사용자가 다음 두 질문에 답하기 어렵게 만든다.

1. OpenHarness를 Agent Harness 관점에서 어떻게 이해하면 되는가
2. local LLM, 특히 Ollama를 붙여 실제로 Claude Code/Codex와 비슷한 감각으로 어디까지 쓸 수 있는가

따라서 이번 작업은 문서화와 실사용 검증을 분리하지 않고, 하나의 onboarding package로 설계한다.

## 2. Goal

사용자가 다음 흐름을 한 번에 통과할 수 있게 만든다.

1. OpenHarness의 역할과 구조를 부담 없이 이해한다
2. Ollama를 local LLM backend로 연결한다
3. 임의의 작업 폴더에서 `oh`를 실행해 실제 작업을 시도한다
4. 왜 그런 동작이 가능한지 request flow와 subsystem 관점에서 이해한다
5. 현재 한계와 troubleshooting 포인트를 현실적으로 파악한다

## 3. Primary User

이 onboarding package의 1차 독자는 다음과 같다.

- Claude Code 또는 Codex를 조금 써본 경험이 있다
- CLI agent의 기본 감각은 있지만 Agent Harness 내부 구조는 깊게 보지 않았다
- 문서만으로 빠르게 감을 잡고, 바로 local workflow에 적용해 보고 싶다

## 4. Scope

### In Scope

- OpenHarness를 Agent Harness로 이해하기 위한 project-centered 문서화
- Ollama 기반 local LLM onboarding
- `oh`를 임의 작업 폴더에서 실행하는 방식 설명
- request flow, session scope, permission model, tool use 설명
- Claude Code/Codex와의 similarity와 difference 정리
- practical troubleshooting과 known limitations 정리

### Out of Scope

- 모든 module/file에 대한 exhaustive reference
- 모든 provider와 auth path 전수 문서화
- `ohmo` 전체 상세 문서화
- plugin authoring 또는 MCP authoring full tutorial
- 전체 API reference 문서

## 5. Design Principles

- README는 overview와 entry 역할을 유지한다
- 새 문서는 README를 복붙하지 않고, 읽기 쉬운 해설과 workflow에 집중한다
- 구조 설명은 directory listing보다 responsibility와 request flow 중심으로 쓴다
- 실사용 문서는 setup만이 아니라 실제 사용 가능 범위와 실패 가능 지점을 같이 적는다
- 문서는 "바로 써보기"와 "왜 이렇게 동작하는지 이해하기"를 연결해야 한다
- 1차 패키지는 balanced onboarding에 최적화하고, 이후 deeper architecture expansion이 가능해야 한다

## 6. Package Structure

### 6.1 Main Entry

`docs/ONBOARDING_OPENHARNESS_OLLAMA.md`

역할:

- 처음 읽는 메인 entry 문서
- OpenHarness를 어떻게 이해해야 하는지 짧게 설명
- Ollama 연결과 첫 실행을 바로 안내
- 실제 작업 예시를 제공
- deeper docs로 연결

### 6.2 Architecture Guide

`docs/OPENHARNESS_ARCHITECTURE.md`

역할:

- OpenHarness 내부 구조를 project-centered 관점에서 설명
- `request -> model -> tool call -> permission -> execution -> result reinjection` 흐름 정리
- 주요 subsystem의 책임과 관계 설명
- 작업 폴더 기준 scope가 어떻게 잡히는지 설명

### 6.3 Operation Notes

`docs/OLLAMA_OPERATION_NOTES.md`

역할:

- Ollama 연동 실전 운영 노트
- model class 선택, `base_url`, `api_format`, auth behavior 설명
- dummy API key 전제 여부
- known limitations, troubleshooting, recommended usage notes 기록

## 7. Content Design

### 7.1 `ONBOARDING_OPENHARNESS_OLLAMA.md`

권장 흐름:

1. OpenHarness를 어떻게 이해하면 되는가
2. Claude Code/Codex와 비슷한 점과 다른 점
3. Ollama를 붙여 local workflow 구성하기
4. 임의 작업 폴더에서 `oh` 실행하기
5. 실제 작업 prompt 예시 2~3개
6. 왜 이런 동작이 가능한가
7. 현실적인 한계
8. 다음에 읽을 문서 링크

핵심 메시지:

- OpenHarness는 model 자체가 아니라 harness다
- local LLM을 backend로 바꿔도 tool-aware coding workflow를 유지할 수 있다
- 실제 usability는 model quality와 tool use reliability에 크게 좌우된다

### 7.2 `OPENHARNESS_ARCHITECTURE.md`

권장 흐름:

1. OpenHarness를 4개 layer로 보는 관점
2. 한 요청의 lifecycle
3. 주요 subsystem 설명
4. 작업 폴더 단위 scope
5. Ollama 연동 시 중요한 config path
6. deeper dive를 위한 source entry points

핵심 subsystem:

- `engine`
- `tools`
- `permissions`
- `prompts`
- `plugins` / `skills`
- `memory`
- `tasks`
- `swarm`
- `ui`
- `config`

설명 원칙:

- file inventory보다 responsibility 중심
- README 반복보다 interpretation 중심
- 실제 코드 entry point와 연결

### 7.3 `OLLAMA_OPERATION_NOTES.md`

권장 흐름:

1. Ollama를 backend로 쓸 때의 전제
2. OpenAI-compatible path로 붙이는 방식
3. recommended profile setup patterns
4. first-run checklist
5. prompt/task suitability
6. known limitations
7. troubleshooting

반드시 포함할 항목:

- `http://localhost:11434/v1` 같은 base URL 예시
- OpenAI-compatible client path 설명
- auth resolution 상 API key string이 필요한 구현 제약
- local model size/class에 따른 practical advice
- tool-use failure나 weak reasoning에 대한 현실적 기대치

## 8. Validation Design

이 패키지는 문서 작성만으로 완료로 보지 않는다. 다음이 실제로 재현되어야 한다.

1. 메인 onboarding 문서만 읽고 OpenHarness의 위치를 설명할 수 있다
2. Ollama를 backend로 연결할 수 있다
3. 임의 작업 폴더에서 `oh`를 실행할 수 있다
4. codebase summary, file search, refactor suggestion 수준의 실제 작업 예시를 실행할 수 있다
5. 문서의 limitation 설명이 실제 관찰과 크게 어긋나지 않는다

## 9. Similarity / Difference Framing

문서에는 OpenHarness를 Claude Code/Codex의 대체품처럼 과장해서 쓰지 않는다.

반드시 다음 framing을 유지한다.

### Similarities

- CLI agent workflow
- tool-enabled coding assistance
- working directory scoped sessions and context
- multi-turn interactive usage
- permission-mediated execution

### Differences

- backend model과 quality를 사용자가 직접 선택한다
- local LLM은 tool use reliability가 떨어질 수 있다
- OpenHarness는 open-source harness architecture 실험 성격이 더 강하다
- 특정 subscription bridge가 아니라 provider/profile abstraction 위에서 동작한다

## 10. Recommended Execution Order

구현 순서는 다음과 같이 가져간다.

1. 실제 코드 기준으로 structure/source mapping 정리
2. Ollama integration path와 current constraints 확인
3. `ONBOARDING_OPENHARNESS_OLLAMA.md` 초안 작성
4. `OPENHARNESS_ARCHITECTURE.md` 작성
5. `OLLAMA_OPERATION_NOTES.md` 작성
6. 실제 local run validation
7. 문서와 실제 동작 차이 보정

## 11. Risks

- local model capability가 문서 기대치를 못 따라갈 수 있다
- 현재 auth/config path가 local-only workflow에 다소 불친절할 수 있다
- README와 새 문서 간 중복이 생기면 유지보수 비용이 올라간다
- project가 빠르게 바뀌면 architecture 문서가 source와 어긋날 수 있다

## 12. Mitigations

- capability claim보다 observed behavior를 기준으로 문서화한다
- limitation과 workaround를 감추지 않는다
- README는 overview에 머무르게 하고, 상세 설명은 새 문서에 둔다
- code reference를 문서 작성 시 명시적으로 추적한다

## 13. Success Criteria

다음 조건을 만족하면 1차 onboarding package가 성공이다.

- 한 명의 사용자가 문서를 읽고 OpenHarness를 Agent Harness로 설명할 수 있다
- Ollama를 붙여 `oh`를 실제 작업 폴더에서 사용할 수 있다
- 문서가 구조 이해와 첫 성공 경험을 모두 제공한다
- 문서 간 책임이 겹치지 않는다
- 이후 deeper architecture 문서 확장이 자연스럽다

## 14. Follow-up Direction

1차 패키지 이후에는 다음 심화 확장이 가능해야 한다.

- request flow deep dive
- provider/auth resolution deep dive
- plugin/skill loading deep dive
- task/swarm/worktree behavior deep dive
- session persistence and project-scoped state deep dive

이 follow-up은 별도 문서나 appendix로 확장하되, 1차 onboarding package의 entry 부담을 늘리지 않는 방향으로 진행한다.
