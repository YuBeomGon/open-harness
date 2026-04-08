# OpenHarness + Ollama Onboarding

## 이 문서의 목적

이 문서는 Claude Code/Codex를 조금 써봤지만 OpenHarness는 처음인 사용자를 위한 첫 진입 문서다.
목표는 두 가지다.

1. OpenHarness를 Agent Harness로 이해한다.
2. Ollama를 local LLM backend로 붙여 실제로 `oh`를 실행해 본다.

## OpenHarness를 어떻게 이해하면 되는가

OpenHarness는 model 자체가 아니다.
OpenHarness는 model 위에 tool execution, permissions, prompt assembly, session storage, plugin/skill loading을 얹는 harness다.

짧게 말하면:

- Claude Code/Codex와 비슷하게 CLI agent처럼 쓸 수 있다.
- backend model은 고정 구독형이 아니라 provider/profile로 바꿔 붙일 수 있다.
- local LLM을 붙이면 private한 로컬 workflow 실험이 가능하다.

## Claude Code/Codex와 비슷한 점과 다른 점

비슷한 점:

- working directory 기준으로 작업한다.
- multi-turn CLI workflow를 제공한다.
- tool call과 permission-mediated execution을 중심으로 동작한다.

다른 점:

- model quality는 사용자가 붙인 backend에 따라 크게 달라진다.
- local LLM에서는 tool use reliability가 더 낮을 수 있다.
- OpenHarness는 open-source harness architecture를 이해하고 확장하는 데 더 적합하다.

## 첫 번째 local run

이 저장소에서 바로 재현하려면 source checkout 기준으로 `uv run oh`를 사용한다.
이미 전역 설치를 끝냈다면 아래의 `uv run oh`를 `oh`로 바꿔도 된다.

```bash
ollama serve
ollama pull qwen2.5-coder:14b
OPENAI_API_KEY=dummy uv run oh \
  --api-format openai \
  --base-url http://localhost:11434/v1 \
  --model qwen2.5-coder:14b \
  -p "Summarize the purpose of this repository in 5 bullets."
```

이 명령이 성공하면 최소한 다음은 확인된 것이다.

- OpenHarness가 OpenAI-compatible path로 Ollama에 연결된다.
- local model이 한 턴짜리 headless coding-assistant request를 처리한다.
- 현재 문서의 핵심 claim이 실제 실행 경로와 맞는다.

## 매일 쓰는 방식: provider profile 등록

한 번성 테스트가 아니라 반복 사용을 원하면 profile을 등록하는 쪽이 낫다.

```bash
uv run oh provider add ollama-local \
  --label "Ollama Local" \
  --provider openai \
  --api-format openai \
  --auth-source openai_api_key \
  --model qwen2.5-coder:14b \
  --base-url http://localhost:11434/v1

uv run oh provider use ollama-local
OPENAI_API_KEY=dummy uv run oh
```

이후에는 같은 shell 세션에서 `OPENAI_API_KEY=dummy`만 유지하면 된다.

## 임의 작업 폴더에서 실행하기

OpenHarness는 현재 working directory를 session과 project-scoped state의 기준으로 삼는다.
따라서 원하는 프로젝트로 이동한 뒤 그 위치에서 실행하면 된다.

```bash
cd /tmp/openharness-demo-project
OPENAI_API_KEY=dummy oh
```

source checkout 상태라면:

```bash
cd /tmp/openharness-demo-project
OPENAI_API_KEY=dummy /data/MyProject/side/harness/study/OpenHarness/.venv/bin/oh
```

## 실제로 해볼 만한 prompt

- `Review this repository and list the top 3 refactors.`
- `List files that define the permission system and explain how they connect.`
- `Summarize this project as if I were familiar with Claude Code but new to OpenHarness.`

## 현실적인 한계

- local LLM의 품질이 낮으면 tool use와 planning 품질도 같이 낮아진다.
- 현재 구현에서는 OpenAI-compatible local backend에도 API key string이 필요하다.
- Claude Code/Codex와 UX가 비슷한 지점은 있지만 완전히 동일한 product가 아니다.

## 다음에 읽을 문서

- [`OPENHARNESS_ARCHITECTURE.md`](./OPENHARNESS_ARCHITECTURE.md)
- [`OLLAMA_OPERATION_NOTES.md`](./OLLAMA_OPERATION_NOTES.md)
