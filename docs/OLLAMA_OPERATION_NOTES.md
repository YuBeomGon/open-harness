# Ollama Operation Notes

## 이 문서의 목적

이 문서는 OpenHarness에 Ollama를 붙일 때 필요한 실전 메모를 정리한다.
setup command만 나열하지 않고, 실제로 어디서 잘 되고 어디서 흔들리는지까지 기록한다.

## 빠른 health check

`ollama serve`는 foreground process다. 다른 terminal이나 background에서 계속 띄운 뒤, 아래 확인 명령을 별도 shell에서 실행한다.

```bash
ollama serve
```

```bash
ollama list
curl -s http://localhost:11434/api/tags
```

정상이라면 다음 중 하나가 보여야 한다.

- `ollama list`에 로컬 model이 보인다.
- `/api/tags` 응답 JSON에 `models` 배열이 들어 있다.

## 가장 짧은 연결 경로

RTX 2080 Ti class machine에서는 `qwen2.5-coder:7b`를 practical first model로 두는 편이 안전하다.
VRAM headroom이 남는다면 다음 단계로 `qwen2.5-coder:14b`를 시도한다.

```bash
ollama pull qwen2.5-coder:7b
```

```bash
OPENAI_API_KEY=dummy uv run --project /data/MyProject/side/harness/study/OpenHarness/.worktrees/openharness-ollama-onboarding oh \
  --api-format openai \
  --base-url http://localhost:11434/v1 \
  --model qwen2.5-coder:7b \
  -p "List files that define the permission system."
```

## 반복 사용을 위한 profile 방식

```bash
uv run --project /data/MyProject/side/harness/study/OpenHarness/.worktrees/openharness-ollama-onboarding oh provider add ollama-local \
  --label "Ollama Local" \
  --provider openai \
  --api-format openai \
  --auth-source openai_api_key \
  --model qwen2.5-coder:7b \
  --base-url http://localhost:11434/v1

uv run --project /data/MyProject/side/harness/study/OpenHarness/.worktrees/openharness-ollama-onboarding oh provider use ollama-local
OPENAI_API_KEY=dummy uv run --project /data/MyProject/side/harness/study/OpenHarness/.worktrees/openharness-ollama-onboarding oh
```

profile-based flow에서는 `-k dummy`만으로는 실행이 안 되었고, `OPENAI_API_KEY=dummy` 환경 변수가 필요했다.
이 방식은 provider profile을 유지하면서 local backend를 붙일 때 가장 재현성이 좋았다.

## 현재 구현에서 중요한 제약

현재 OpenAI-compatible path는 local backend라도 API key string을 요구한다.
실제로 인증을 하지 않는 Ollama를 붙일 때도 `OPENAI_API_KEY=dummy` 같은 값을 넣어야 한다.

이 제약은 문서에서 숨기지 말고 명시한다.

또한 `codellama:13b`는 테스트 중 Ollama OpenAI-compatible path에서 `does not support tools`를 반환했다.
Claude-Code-like tool use를 기대할 때는 이 모델을 practical option으로 보지 않는 편이 낫다.

## 권장 usage

- repository summary
- file discovery
- permission and architecture explanation
- low-risk refactor brainstorming

`qwen2.5-coder:7b`는 backend 연결과 basic prompt handling을 확인하는 데는 유효했지만, tool use와 workspace inspection 품질은 약했고 현재 directory/files를 잘못 추론하는 경우가 있었다.
local model quality가 충분하지 않다면, large patch generation이나 복잡한 multi-step tool planning은 기대치를 낮춰야 한다.

## Troubleshooting

### 증상: `No credentials found for auth source 'openai_api_key'`

원인:
- OpenAI-compatible path가 빈 API key를 허용하지 않는다.

대응:

```bash
export OPENAI_API_KEY=dummy
```

### 증상: `Connection error` 또는 `localhost:11434` timeout

원인:
- Ollama daemon이 떠 있지 않거나 model pull이 끝나지 않았다.

대응:

```bash
ollama serve
ollama list
curl -s http://localhost:11434/api/tags
```

### 증상: 답변은 오지만 tool use가 약하다

원인:
- local model의 coding/tool-use 품질 한계다.

대응:
- prompt를 더 짧고 명시적으로 쓴다.
- one-shot summarization이나 code reading 위주로 사용한다.
- model class를 더 강한 coding model로 올린다.

### 증상: `codellama:13b`가 tools를 지원하지 않는다고 나온다

원인:
- Ollama OpenAI-compatible path에서 해당 model이 tools-capable로 노출되지 않았다.

대응:
- `qwen2.5-coder:7b`부터 다시 확인한다.
- VRAM headroom이 있으면 `qwen2.5-coder:14b`를 시도한다.

## Observed limitations

- OpenHarness는 Ollama를 OpenAI-compatible backend로 취급한다.
- local model 품질이 낮으면 Claude Code/Codex와 비슷한 workflow라도 체감 품질 차이가 크다.
- 문서의 목표는 “동일 product”가 아니라 “비슷한 harness workflow”를 재현하는 것이다.
