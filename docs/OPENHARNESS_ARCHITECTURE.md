# OpenHarness Architecture Guide

## 이 문서의 목적

이 문서는 OpenHarness를 directory tree가 아니라 request flow와 subsystem responsibility 중심으로 이해하기 위한 문서다.
코드를 전부 따라가지 않고도, 어떤 입력이 어떤 계층을 거쳐 실행되는지 파악하는 데 초점을 둔다.

## OpenHarness를 4개 layer로 보는 관점

1. UI layer
   사용자가 입력하고 결과를 보는 layer다. CLI entry, React/Ink TUI, backend host가 여기에 들어간다.
2. Runtime layer
   provider/auth resolution, tool registry 구성, hook/plugin loading, `QueryEngine` 구성이 여기서 일어난다.
3. Agent loop layer
   model response, tool call, permission check, result reinjection이 반복되는 핵심 loop다.
4. Persistence and extension layer
   sessions, project-scoped `.openharness/`, skills, plugins, memory, tasks, swarm 기능이 여기에 들어간다.

## 주요 subsystem

- `src/openharness/cli.py`
- `src/openharness/ui/app.py`
- `src/openharness/ui/runtime.py`
- `src/openharness/engine/query_engine.py`
- `src/openharness/tools/__init__.py`
- `src/openharness/permissions/checker.py`
- `src/openharness/services/session_storage.py`
- `src/openharness/config/paths.py`

## 한 요청의 lifecycle

1. `src/openharness/cli.py`가 CLI option을 파싱하고 interactive mode 또는 print mode로 진입한다.
2. `src/openharness/ui/app.py`가 TUI launch 또는 headless 실행 경로를 고른다.
3. `src/openharness/ui/runtime.py`가 settings override를 머지하고 provider/auth를 resolve한 뒤 `RuntimeBundle`을 만든다.
4. `src/openharness/engine/query_engine.py`가 user message를 conversation에 추가하고 tool-aware query loop를 시작한다.
5. model이 tool call을 반환하면 `src/openharness/tools/__init__.py`에서 등록된 tool registry를 통해 실행 경로가 선택된다.
6. mutating action은 `src/openharness/permissions/checker.py`를 통과해야 한다.
7. 결과는 다시 conversation에 주입되고, 필요하면 다음 model turn으로 이어진다.
8. 세션 스냅샷은 `src/openharness/services/session_storage.py`를 통해 project-scoped 형태로 저장된다.

## 작업 폴더 단위 scope

OpenHarness는 현재 working directory를 단순한 shell 위치가 아니라 project boundary로 취급한다.
이 경계는 최소한 다음 항목들에 영향을 준다.

- session snapshot grouping
- project-local `.openharness/` directory
- CLAUDE.md discovery
- plugin visibility
- memory and issue-context lookup

즉 `cd /tmp/openharness-demo-project && oh`는 그 프로젝트를 기준으로 context와 state를 묶는 동작이다.

## subsystem responsibility 요약

### `engine`

`QueryEngine`는 대화 기록, max turn, system prompt, API client, tool registry를 묶고 실제 tool-aware loop를 호출한다.

### `tools`

bash, file I/O, search, MCP, tasks, agent/team 관련 built-in tool이 registry에 등록된다.

### `permissions`

sensitive path deny, read-only allow, plan mode block, default confirmation 같은 safety 정책이 여기서 적용된다.

### `config`

settings, profile materialization, path resolution, runtime defaults/overrides를 담당한다.

### `prompts`

system prompt, environment info, CLAUDE.md, memory, skill registry가 한 runtime prompt로 합쳐진다.

### `plugins` / `skills`

plugins는 user-global과 project-local plugin directory를 읽어 extension surface를 구성한다.
skills는 bundled skills, configured config dir의 user skills (`<config-dir>/skills`), 그리고 plugin loading을 통해 발견되는 plugin-provided skills를 합쳐 노출한다.

### `memory`, `tasks`, `swarm`

long-lived context, background execution, subagent-style task delegation을 담당한다.

### `ui`

React/Ink frontend와 backend host가 interactive experience를 만든다.

## Ollama를 붙일 때 중요한 경로

Ollama는 현재 codebase에서 OpenAI-compatible local backend로 취급된다.
핵심 경로는 다음과 같다.

- `src/openharness/config/settings.py`: active profile, `api_format`, `base_url`, auth resolution
- `src/openharness/ui/runtime.py`: resolved settings를 바탕으로 `OpenAICompatibleClient` 생성
- `src/openharness/api/openai_client.py`: OpenAI-compatible streaming path
- `src/openharness/api/registry.py`: `http://localhost:11434/v1` 같은 local backend 식별

## deeper dive를 위한 source entry points

다음 파일은 doc-to-source alignment를 유지하기 위한 starting set이다. request flow와 책임 분리를 더 좁혀 보려면 아래부터 열어보면 된다.

- `src/openharness/cli.py`
- `src/openharness/ui/runtime.py`
- `src/openharness/engine/query_engine.py`
- `src/openharness/engine/query.py`
- `src/openharness/tools/__init__.py`
- `src/openharness/permissions/checker.py`
- `src/openharness/prompts/context.py`
- `src/openharness/plugins/loader.py`
- `src/openharness/skills/loader.py`
- `src/openharness/config/settings.py`
- `src/openharness/services/session_storage.py`

## 이 문서를 검증할 때 사용한 source checks

이 checks는 code가 바뀔 때 doc-to-source alignment를 유지하기 위한 starting set이다.

- `rg -n "provider add|provider use|--base-url|--api-format" src/openharness/cli.py`
- `rg -n "OpenAICompatibleClient|build_runtime|cwd = str\\(Path.cwd\\(\\)\\)" src/openharness/ui/runtime.py`
- `rg -n "QueryEngine|submit_message|continue_pending" src/openharness/engine/query_engine.py`
- `rg -n "SENSITIVE_PATH_PATTERNS|PermissionDecision" src/openharness/permissions/checker.py`
- `rg -n "localhost:11434|Ollama" src/openharness/api/registry.py`
