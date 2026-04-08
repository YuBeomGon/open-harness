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

이 문서는 두 가지 launch mode를 구분한다.

- source checkout validation
- installed CLI를 arbitrary working dir에서 쓰는 방식

source checkout만 있는 상태에서 `uv run`으로 `oh`를 어떤 폴더에서든 바로 부르면 `Failed to spawn: oh`가 났다.
따라서 source checkout 기준 검증은 `uv run --project <path-to-OpenHarness-checkout> oh ...`로 한다.
이 문서의 source checkout 예시는 launcher checkout을 `--project <path-to-OpenHarness-checkout>`로 지정하고, prompt는 현재 working directory에서 해석되는 방식으로 검증했다.
다른 프로젝트를 검사하려면 먼저 `cd /path/to/target-project` 한 뒤 실행한다.

이 문서의 source checkout 예시는 모두 `<path-to-OpenHarness-checkout>`를 실제 checkout root로 바꿔서 실행한다.
installed CLI가 있으면 같은 명령에서 `uv run --project <path-to-OpenHarness-checkout> oh` 부분을 `oh`로 바꿔서 쓴다.

먼저 `ollama serve`를 별도 terminal에서 계속 실행하거나 background로 둔다.

```bash
ollama serve
```

같은 Ollama instance를 대상으로 model을 내려받는다.

```bash
ollama pull qwen2.5-coder:7b
```

source checkout 기준의 첫 검증은 다음처럼 한다.

```bash
OPENAI_API_KEY=dummy uv run --project <path-to-OpenHarness-checkout> oh \
  --api-format openai \
  --base-url http://localhost:11434/v1 \
  --model qwen2.5-coder:7b \
  -p "Summarize the purpose of the current workspace in 5 bullets."
```

이 명령이 성공하면 최소한 다음은 확인된 것이다.

- OpenHarness가 OpenAI-compatible path로 Ollama에 연결된다.
- local model이 한 턴짜리 headless coding-assistant request를 처리한다.
- `qwen2.5-coder:7b` 기준으로 backend connection path가 실제로 동작한다.

installed CLI를 이미 준비한 상태라면, `/tmp/openharness-demo-project` 같은 arbitrary working dir에서도 `oh`를 직접 실행할 수 있다.
이 문서에서 확인한 source checkout 흐름은 `uv run --project <path-to-OpenHarness-checkout> oh ...`였다.
`oh`만으로 Ollama로 가는 것은 `oh provider add`와 `oh provider use`로 provider profile을 이미 활성화한 뒤에만 성립한다.
profile을 아직 등록하지 않았다면 source-checkout 예시처럼 `--api-format openai --base-url http://localhost:11434/v1 --model qwen2.5-coder:7b`를 직접 넘겨야 한다.

## 매일 쓰는 방식: provider profile 등록

한 번성 테스트가 아니라 반복 사용을 원하면 profile을 등록하는 쪽이 낫다.

```bash
uv run --project <path-to-OpenHarness-checkout> oh provider add ollama-local \
  --label "Ollama Local" \
  --provider openai \
  --api-format openai \
  --auth-source openai_api_key \
  --model qwen2.5-coder:7b \
  --base-url http://localhost:11434/v1

uv run --project <path-to-OpenHarness-checkout> oh provider use ollama-local
OPENAI_API_KEY=dummy uv run --project <path-to-OpenHarness-checkout> oh
```

installed CLI라면 위의 `uv run --project <path-to-OpenHarness-checkout> oh ...` 앞부분을 모두 `oh ...`로 바꾼다.
이 흐름에서도 `oh`만 쓰는 경우는 provider profile이 `use`로 활성화된 상태를 전제로 한다.
profile을 쓰지 않으면 `OPENAI_API_KEY=dummy`만으로는 충분하지 않고, Ollama용 flags를 명시해야 한다.

profile-based flow에서는 `-k dummy`만으로는 충분하지 않았고, `OPENAI_API_KEY=dummy` 환경 변수가 필요했다.
반복 실행할 때는 이 env var를 기본값처럼 두는 편이 안전하다.

## 임의 작업 폴더에서 실행하기

OpenHarness는 현재 working directory를 session과 project-scoped state의 기준으로 삼는다.
따라서 installed CLI가 있다면 원하는 프로젝트로 이동한 뒤 그 위치에서 실행하면 된다.

```bash
cd /tmp/openharness-demo-project
OPENAI_API_KEY=dummy oh
```

source checkout validation을 유지하려면 위에서 설명한 `uv run --project <path-to-OpenHarness-checkout> oh ...` 패턴을 그대로 쓴다.

## 실제로 해볼 만한 prompt

- `Review this repository and list the top 3 refactors.`
- `List files that define the permission system and explain how they connect.`
- `Summarize this project as if I were familiar with Claude Code but new to OpenHarness.`

## 현실적인 한계

- local LLM의 품질이 낮으면 tool use와 planning 품질도 같이 낮아진다.
- `qwen2.5-coder:7b`는 backend connection path를 검증하기에는 충분했지만, tool use와 workspace inspection 품질은 약했고 현재 directory/files를 잘못 추론하는 hallucination도 보였다.
- 현재 구현에서는 OpenAI-compatible local backend에도 API key string이 필요하다.
- Claude Code/Codex와 UX가 비슷한 지점은 있지만 완전히 동일한 product가 아니다.

## 다음에 읽을 문서

이 onboarding package의 companion 문서는 layered package로 묶어서 읽는 것이 좋다.

- `OPENHARNESS_ARCHITECTURE.md`: OpenHarness의 layer, request flow, subsystem responsibility를 먼저 잡는 architecture guide다.
- `OLLAMA_OPERATION_NOTES.md`: Ollama 연결, validation caveat, troubleshooting을 정리한 operation notes다.

먼저 `OPENHARNESS_ARCHITECTURE.md`로 구조를 잡고, 그 다음 `OLLAMA_OPERATION_NOTES.md`로 실행 메모를 확인하면 된다.
