HKUDS · MIT License · v0.1.2

# oh — OpenHarness

경량 오픈소스 Agent 인프라 프레임워크. Tool-use, Skills, Memory, Multi-Agent Coordination을 하나의 커맨드로.

Python ≥ 3.10 43+ Tools 114 Tests Passing React/Ink TUI

## 01 Agent Harness란?

**Agent Harness**는 LLM을 실질적인 에이전트로 만들기 위해 감싸는 인프라 계층입니다. 모델은 *지능*을, Harness는 **손(도구)**, **눈(관찰)**, **기억(메모리)**, **안전 경계(권한)**를 제공합니다.

**핵심 등식:** Agent = LLM + Tools + Knowledge + Observation + Action + Permissions

OpenHarness는 Claude Code의 harness 구조(CLAUDE.md, skills, hooks, /compact 등)를 Python으로 오픈소스 재구현한 프로젝트입니다. OpenClaw, nanobot, Cursor 등 다른 CLI 에이전트와도 통합 가능하도록 설계되었습니다.

- **Understand** — 프로덕션 AI 에이전트가 내부적으로 어떻게 동작하는지 학습
- **Experiment** — 최신 도구, 스킬, 에이전트 조율 패턴 실험
- **Extend** — 커스텀 플러그인, 프로바이더, 도메인 지식으로 확장
- **Build** — 검증된 아키텍처 위에 특화 에이전트 구축

## 02 5대 핵심 기능

🔄

#### Agent Loop

스트리밍 Tool-Call 사이클, 지수 백오프 재시도 (MAX_RETRIES=3, 429/500/502/503/529), 병렬 도구 실행, CostTracker 토큰 추적

🔧

#### Harness Toolkit

43+ 내장 도구 (File I/O, Shell, Search, Web, MCP, Agent, Task, Cron). Pydantic 입력 검증, JSON Schema 자동 생성

🧠

#### Context & Memory

CLAUDE.md 탐색·주입, Auto-Compact (microcompact + LLM 요약), MEMORY.md 영속 메모리, 세션 Resume

🛡️

#### Governance

3단계 권한 모드, 경로 glob 규칙, 커맨드 거부 리스트, Pre/PostToolUse 훅, 센시티브 경로 자동 차단

🤝

#### Swarm Coordination

서브에이전트 스폰 (AgentTool), TeamRegistry, 백그라운드 태스크 (create/get/list/update/stop/output)

🖥️

#### React TUI

React/Ink 터미널 UI — 커맨드 피커, 권한 다이얼로그, 세션 Resume, 키보드 단축키, 스피너

## 03 설치 & 설정

### 사전 요구사항

- **Python 3.10+** 및 `uv` (astral.sh)
- **Node.js 18+** (선택 — React TUI용)
- LLM API Key (Anthropic, OpenAI, Kimi 등)

### 원클릭 설치

    curl -fsSL https://raw.githubusercontent.com/HKUDS/OpenHarness/main/scripts/install.sh | bash

스크립트가 자동으로: OS 감지 → Python/Node 버전 검증 → pip 설치 → TUI npm install → `~/.openharness/` 생성 → `oh --version` 확인.

| 플래그            | 설명                                                    |
|-------------------|---------------------------------------------------------|
| `--from-source`   | GitHub 클론 후 `pip install -e .`                       |
| `--with-channels` | IM 채널 의존성 함께 설치 (slack-sdk, telegram, discord) |

### 수동 설치

    git clone https://github.com/HKUDS/OpenHarness.git
    cd OpenHarness
    uv sync --extra dev

    # Quick Demo
    ANTHROPIC_API_KEY=your_key uv run oh -p "Inspect this repo and list top 3 refactors"

    # Kimi 백엔드 예시
    export ANTHROPIC_BASE_URL=https://api.moonshot.cn/anthropic
    export ANTHROPIC_API_KEY=your_kimi_key
    export ANTHROPIC_MODEL=kimi-k2.5
    uv run oh

### Setup Workflow

`oh setup`은 대화형 워크플로우 기반 설정입니다:

1.  워크플로우 선택 (Anthropic-Compatible / Claude Subscription / OpenAI-Compatible / Codex / Copilot)
2.  백엔드 프리셋 선택 (Claude, Kimi, GLM, MiniMax 등)
3.  인증 → 모델 선택 → 프로파일 저장 & 활성화

<!-- -->

    uv run oh setup              # 대화형 설정
    oh provider list             # 프로파일 목록
    oh provider use codex        # 프로파일 전환

### Provider 호환성

| 워크플로우               | 포맷      | 백엔드 예시                                           |
|--------------------------|-----------|-------------------------------------------------------|
| **Anthropic-Compatible** | Anthropic | Claude, Kimi, GLM, MiniMax                            |
| **Claude Subscription**  | Bridge    | ~/.claude/.credentials.json                           |
| **OpenAI-Compatible**    | OpenAI    | OpenAI, OpenRouter, DashScope, DeepSeek, Groq, Ollama |
| **Codex Subscription**   | Bridge    | ~/.codex/auth.json                                    |
| **GitHub Copilot**       | OAuth     | GitHub Device Flow                                    |

`oh provider add`로 커스텀 엔드포인트 등록 시 프로파일별 크레덴셜 분리 가능.

### 환경변수

| 변수                     | 용도                                    |
|--------------------------|-----------------------------------------|
| `ANTHROPIC_API_KEY`      | Anthropic / Anthropic-Compatible API 키 |
| `ANTHROPIC_BASE_URL`     | 커스텀 엔드포인트 URL                   |
| `ANTHROPIC_MODEL`        | 모델 지정                               |
| `OPENAI_API_KEY`         | OpenAI-Compatible 사용 시               |
| `OPENHARNESS_API_FORMAT` | anthropic / openai / copilot            |

## 04 아키텍처 — 코드 레벨 분석

### 디렉토리 구조

src/openharness/ ├─ engine/Agent Loop — query_engine.py, query.py, messages.py, stream_events.py, cost_tracker.py ├─ api/API 클라이언트 — client.py (Anthropic), openai_client.py, copilot_client.py, codex_client.py ├─ tools/43개 도구 — base.py (BaseTool, ToolRegistry) + 각 \*\_tool.py ├─ permissions/modes.py (3 PermissionMode) + checker.py (PermissionChecker) ├─ hooks/schemas.py (4 Hook타입) + executor.py (HookExecutor) + loader.py, hot_reload.py ├─ skills/loader.py, registry.py, types.py — .md 기반 on-demand 지식 ├─ plugins/loader.py — claude-code 플러그인 호환 ├─ memory/memdir.py, manager.py, scan.py, search.py, paths.py, types.py ├─ prompts/system_prompt.py, claudemd.py, context.py, environment.py ├─ services/compact/ (auto-compact), token_estimation.py, session_storage.py, cron.py ├─ coordinator/TeamRegistry, TeamRecord, agent_definitions.py ├─ tasks/manager.py, local_agent_task.py, local_shell_task.py ├─ config/settings.py (Pydantic), paths.py, schema.py ├─ mcp/client.py, config.py, types.py ├─ auth/manager.py, external.py, storage.py, flows.py ├─ ui/React TUI backend + frontend ├─ bridge/CLI subscription bridge (Claude/Codex) ├─ commands/54 slash commands ├─ sandbox/OS-level sandboxing ├─ voice/음성 입력 (STT) └─ themes/TUI 테마 시스템

### engine/ — Agent Loop (심장부)

모델이 **무엇**을 할지 결정하면, engine이 **어떻게** — 안전하고 효율적으로 — 실행합니다.

#### QueryEngine (query_engine.py)

    class QueryEngine:
        """대화 히스토리 소유 + tool-aware 모델 루프 관리"""

        def __init__(self, *,
            api_client: SupportsStreamingMessages,   # Protocol — 테스트 mock 가능
            tool_registry: ToolRegistry,              # 등록된 43+ 도구
            permission_checker: PermissionChecker,    # 권한 검사기
            cwd: str | Path,
            model: str,
            system_prompt: str,
            max_tokens: int = 4096,
            max_turns: int | None = 8,                # 한 입력당 최대 턴
            hook_executor: HookExecutor | None = None,
        ):
            self._messages: list[ConversationMessage] = []
            self._cost_tracker = CostTracker()

#### run_query (query.py) — 핵심 루프

    async def run_query(context: QueryContext, messages) -> AsyncIterator[...]:
        """Auto-compaction → API stream → tool 실행 → 루프"""
        compact_state = AutoCompactState()
        turn_count = 0

        while context.max_turns is None or turn_count < context.max_turns:
            turn_count += 1

            # ① Auto-compact check (microcompact → full LLM 요약)
            messages, was_compacted = await auto_compact_if_needed(...)

            # ② API 스트리밍 호출
            async for event in context.api_client.stream_message(...):
                if isinstance(event, ApiTextDeltaEvent):
                    yield AssistantTextDelta(text=event.text), None
                elif isinstance(event, ApiRetryEvent):
                    yield StatusEvent(f"Retrying in {event.delay_seconds}s...")

            # ③ stop_reason != "tool_use" → 종료
            if stop_reason != "tool_use":
                yield AssistantTurnComplete(...); return

            # ④ Permission → Hook → Execute → Hook → Result
            for tool_call in tool_uses:
                result = await _execute_single_tool(tool_call, context)

            messages.append(tool_results)  # 루프 계속

#### 실행 흐름

입력User Prompt → 인터페이스CLI / TUI → 엔진QueryEngine → APIstream_message → 분기tool_use? → 검증Perm+Hook → 실행Tool.execute ↩

#### StreamEvent 타입 (stream_events.py)

| 이벤트                             | 역할                               |
|------------------------------------|------------------------------------|
| `AssistantTextDelta`               | 모델 텍스트 청크 (스트리밍)        |
| `AssistantTurnComplete`            | 모델 턴 종료 — 전체 메시지 + usage |
| `ToolExecutionStarted / Completed` | 도구 실행 시작/완료                |
| `StatusEvent`                      | 재시도, compact 등 상태            |
| `ErrorEvent`                       | 오류 발생                          |

### api/ — API 클라이언트

    class SupportsStreamingMessages(Protocol):
        """테스트 mock / 다중 프로바이더 호환용 Protocol"""
        async def stream_message(self, req: ApiMessageRequest) -> AsyncIterator[ApiStreamEvent]: ...

    # 구현체 4개:
    # client.py        → AsyncAnthropic 래퍼 (Anthropic-Compatible)
    # openai_client.py → OpenAI /v1/chat/completions
    # copilot_client.py → GitHub Copilot OAuth
    # codex_client.py  → Codex subscription bridge

**Retry:** MAX_RETRIES=3, BASE_DELAY=1s, MAX_DELAY=30s, 지수 백오프. 대상: `{429, 500, 502, 503, 529}`

### tools/ — 도구 시스템

#### BaseTool & ToolRegistry (base.py)

    class BaseTool(ABC):
        name: str
        description: str
        input_model: type[BaseModel]         # Pydantic → 자동 JSON Schema

        async def execute(self, args, ctx: ToolExecutionContext) -> ToolResult: ...
        def is_read_only(self, args) -> bool: ...  # 권한 판단용
        def to_api_schema(self) -> dict: ...       # Anthropic API 형식 변환

    class ToolRegistry:
        def register(self, tool): ...
        def get(self, name) -> BaseTool | None: ...
        def to_api_schema(self) -> list[dict]: ...  # 전체 도구 → API 전달

#### 전체 도구 목록 (43+)

| 카테고리     | 도구                                               | 설명                      |
|--------------|----------------------------------------------------|---------------------------|
| **File I/O** | bash, read_file, write_file, edit_file, glob, grep | 파일 조작 + 권한 검사     |
| **Search**   | web_fetch, web_search, tool_search, lsp            | 웹·코드 검색              |
| **Notebook** | notebook_edit                                      | Jupyter 셀 편집           |
| **Agent**    | agent, send_message, team_create/delete            | 서브에이전트 스폰·협업    |
| **Task**     | task_create/get/list/update/stop/output            | 백그라운드 태스크         |
| **MCP**      | mcp_tool, list/read_mcp_resource, mcp_auth         | Model Context Protocol    |
| **Mode**     | enter/exit_plan_mode, enter/exit_worktree          | 워크플로우 모드 전환      |
| **Cron**     | cron_create/list/delete/toggle, remote_trigger     | 스케줄 실행               |
| **Meta**     | skill, config, brief, sleep, ask_user, todo_write  | 지식 로딩, 설정, 상호작용 |

### permissions/ — 권한 체커

#### 3단계 PermissionMode

| 모드        | 동작                     | 용도          |
|-------------|--------------------------|---------------|
| `DEFAULT`   | 쓰기/실행 전 사용자 확인 | 일상 개발     |
| `FULL_AUTO` | 모든 것 자동 허용        | 샌드박스      |
| `PLAN`      | 모든 쓰기 차단           | 리팩토링 검토 |

#### PermissionChecker (checker.py)

    # 하드코딩된 센시티브 경로 — 항상 거부 (프롬프트 인젝션 방어)
    SENSITIVE_PATH_PATTERNS = (
        "*/.ssh/*", "*/.aws/credentials", "*/.aws/config",
        "*/.config/gcloud/*", "*/.azure/*", "*/.gnupg/*",
        "*/.docker/config.json", "*/.kube/config",
        "*/.openharness/credentials.json",
        "*/.openharness/copilot_auth.json",
    )

    class PermissionChecker:
        def evaluate(self, tool_name, *, is_read_only, file_path=None) -> PermissionDecision:
            # 1. 센시티브 경로 fnmatch → 무조건 거부
            # 2. denied_commands / denied_tools → 거부
            # 3. path_rules (glob) → allow/deny
            # 4. PermissionMode → allowed / requires_confirmation

settings.json 예시:

    {
      "permission": {
        "mode": "default",
        "path_rules": [{ "pattern": "/etc/*", "allow": false }],
        "denied_commands": ["rm -rf /", "DROP TABLE *"]
      }
    }

### hooks/ — 라이프사이클 훅

#### 4가지 Hook 타입 (schemas.py)

| 타입                    | 동작                    | timeout | block_on_failure |
|-------------------------|-------------------------|---------|------------------|
| `CommandHookDefinition` | 셸 커맨드 실행          | 30s     | false            |
| `PromptHookDefinition`  | 모델에게 조건 검증      | 30s     | true             |
| `HttpHookDefinition`    | HTTP POST 페이로드 전송 | 30s     | false            |
| `AgentHookDefinition`   | 모델 기반 심화 검증     | 60s     | true             |

    class HookExecutor:
        async def execute(self, event: HookEvent, payload) -> AggregatedHookResult:
            # event = PreToolUse | PostToolUse
            for hook in self._registry.get(event):
                if _matches_hook(hook, payload):
                    # hook.type에 따라 command/http/prompt/agent 실행
                    # block_on_failure=True면 도구 실행 차단 가능

**보안:** command hook의 `$ARGUMENTS` 치환 시 shell-escape 적용. `$(…)` / backtick 통한 injection 방어.

### skills/ — 스킬 로더

    def load_skill_registry(cwd, *, extra_skill_dirs, extra_plugin_roots):
        registry = SkillRegistry()
        # 1. 번들 스킬 (내장)
        for skill in get_bundled_skills(): registry.register(skill)
        # 2. 사용자 스킬 (~/.openharness/skills/*.md)
        for skill in load_user_skills(): registry.register(skill)
        # 3. 추가 디렉토리 (extra_skill_dirs)
        # 4. 활성 플러그인에 포함된 스킬
        return registry

**anthropics/skills 호환** — `~/.openharness/skills/`에 .md 복사만으로 사용 가능.

### memory/ — 영속 메모리

#### 경로 설계 (paths.py)

    def get_project_memory_dir(cwd) -> Path:
        path = Path(cwd).resolve()
        digest = sha1(str(path).encode()).hexdigest()[:12]
        # ~/.openharness/data/memory/{project_name}-{hash12}/
        return get_data_dir() / "memory" / f"{path.name}-{digest}"

#### 관리 API (manager.py)

- `add_memory_entry(cwd, title, content)` → slug 파일 + MEMORY.md 인덱스
- `remove_memory_entry(cwd, name)` → 파일 삭제 + 인덱스 제거
- `list_memory_files(cwd)` → 전체 목록

#### 검색 (search.py)

    def find_relevant_memories(query, cwd, *, max_results=5):
        # frontmatter (title, description) 매치: 가중치 2x
        # body content 매치: 가중치 1x
        # Han character 토크나이저 → 한중일 다국어 지원

### prompts/ — 프롬프트 조립

#### 시스템 프롬프트 핵심 원칙 (system_prompt.py)

- 전용 도구 우선 (read_file \> cat, edit_file \> sed, Bash는 셸 명령 전용)
- 안전 코딩 (OWASP, injection 방지)
- 최소 변경 ("요청된 것만 수정, 추가 개선 금지")
- 가역성 판단 ("돌이킬 수 없는 작업은 사용자 확인")
- 간결 출력 ("한 문장이면 세 문장 쓰지 말 것")

#### CLAUDE.md 탐색 (claudemd.py)

    def discover_claude_md_files(cwd) -> list[Path]:
        # cwd → 상위 디렉토리까지 순회하며:
        #   {dir}/CLAUDE.md
        #   {dir}/.claude/CLAUDE.md
        #   {dir}/.claude/rules/*.md (정렬됨)
        # max_chars_per_file = 12,000 (초과 시 truncate)

### services/compact/ — Auto-Compact

Claude Code의 compaction을 충실히 재구현. 컨텍스트 윈도우 자동 관리:

| 단계             | 방법                                                     | 비용    |
|------------------|----------------------------------------------------------|---------|
| **Microcompact** | 오래된 tool result → `[Old tool result content cleared]` | 무료    |
| **Full Compact** | LLM으로 오래된 메시지를 구조화 요약                      | API 1회 |

    COMPACTABLE_TOOLS = frozenset({"read_file", "bash", "grep", "glob",
        "web_search", "web_fetch", "edit_file", "write_file"})
    AUTOCOMPACT_BUFFER_TOKENS    = 13_000
    MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000
    MAX_CONSECUTIVE_FAILURES      = 3
    DEFAULT_CONTEXT_WINDOW        = 200_000
    TOKEN_ESTIMATION_PADDING      = 4 / 3  # 보수적 추정 (len(text)//4 기반)

### coordinator/ — Multi-Agent

    class TeamRegistry:
        def create_team(self, name, description) -> TeamRecord: ...
        def add_agent(self, team_name, task_id): ...
        def send_message(self, team_name, message): ...
        def list_teams(self) -> list[TeamRecord]: ...

    @dataclass
    class TeamRecord:
        name: str; description: str
        agents: list[str]    # task_id 목록
        messages: list[str]  # 팀 내 메시지 히스토리

### config/ — 설정 스키마 (settings.py)

우선순위: **CLI → 환경변수 → settings.json → 기본값**

    class PermissionSettings(BaseModel):
        mode: PermissionMode = DEFAULT
        allowed_tools / denied_tools: list[str]
        path_rules: list[PathRuleConfig]
        denied_commands: list[str]

    class MemorySettings(BaseModel):
        enabled: bool = True; max_files: int = 5

    class SandboxSettings(BaseModel):
        enabled: bool = False
        network: SandboxNetworkSettings     # allowed/denied domains
        filesystem: SandboxFilesystemSettings # allow/deny read/write

    class ProviderProfile(BaseModel):
        label: str; provider: str; ...       # 프로파일별 크레덴셜 분리

## 05 CLI 레퍼런스

### 옵션 & 플래그

    oh [OPTIONS] COMMAND [ARGS]

| 카테고리                         | 플래그                      | 설명                       |
|----------------------------------|-----------------------------|----------------------------|
| **Session**                      | `-c, --continue`            | 이전 대화 이어서           |
| `-r, --resume`                   | 세션 이력에서 복원          |                            |
| `-n, --name`                     | 세션 이름 지정              |                            |
| **Model**                        | `-m, --model`               | 모델 지정                  |
| `--effort`                       | 토큰 예산 수준              |                            |
| `--max-turns`                    | 최대 에이전트 턴            |                            |
| **Output**                       | `-p, --print`               | 비대화형 stdout            |
| `--output-format`                | text \| json \| stream-json |                            |
| **Permission**                   | `--permission-mode`         | default / full_auto / plan |
| `--dangerously-skip-permissions` | 권한 검사 건너뛰기          |                            |
| **Context**                      | `-s, --system-prompt`       | 시스템 프롬프트 교체       |
| `--append-system-prompt`         | 기본에 추가                 |                            |
| **Advanced**                     | `-d, --debug`               | 디버그 로깅                |
| `--mcp-config`                   | MCP 설정 경로               |                            |
| `--bare`                         | 최소 설정 실행              |                            |

### 서브커맨드

| 커맨드                                        | 설명                   |
|-----------------------------------------------|------------------------|
| `oh setup`                                    | 대화형 워크플로우 설정 |
| `oh provider list|use|add`                    | 프로파일 관리          |
| `oh auth copilot-login|status|copilot-logout` | Copilot OAuth          |
| `oh mcp ...`                                  | MCP 서버 관리          |
| `oh plugin list|install|enable`               | 플러그인 관리          |

### 비대화형 모드

    oh -p "Explain this codebase"                           # text → stdout
    oh -p "List functions" --output-format json              # JSON 출력
    oh -p "Fix the bug" --output-format stream-json          # 실시간 스트림

## 06 ohmo — Personal Agent

OpenHarness 위에 구축된 개인 에이전트 앱. 자체 워크스페이스, 게이트웨이, 인격(soul) 시스템.

    ohmo init       # 워크스페이스 초기화 (~/.ohmo/)
    ohmo config     # 게이트웨이 채널 + 프로바이더 설정
    ohmo            # 에이전트 실행
    ohmo gateway run|status|restart

~/.ohmo/ ├─ soul.md── 에이전트 인격 & 행동 원칙 ├─ identity.md── 정체성 ├─ user.md── 사용자 프로필 & 선호 ├─ BOOTSTRAP.md── 첫 실행 리추얼 ├─ memory/── 영속 메모리 ├─ state.json── 최근 상태 └─ gateway.json── 프로바이더 + 채널 설정

#### SOUL.md 핵심 (workspace.py)

- "제네릭 어시스턴트처럼 들리지 마라" — 유용하고 신뢰할 수 있는 존재
- 판단력 — 선호를 가지고 트레이드오프 설명
- 물어보기 전에 스스로 해결 — 파일 읽기, 컨텍스트 점검
- "접근은 친밀함" — 메시지·파일·노트를 존중

### 코드 구조 (ohmo/)

| 파일               | LOC | 역할                                        |
|--------------------|-----|---------------------------------------------|
| cli.py             | 614 | CLI 진입점 (init, config, gateway, main)    |
| workspace.py       | 307 | 워크스페이스 초기화 + 템플릿                |
| runtime.py         | 192 | OpenHarness runtime 연결 (run_ohmo_backend) |
| session_storage.py | 192 | 세션 저장/복원                              |
| prompts.py         | 74  | 시스템 프롬프트 빌더                        |
| memory.py          | 69  | 개인 메모리 헬퍼                            |

지원 채널: Telegram, Slack, Discord, Feishu

## 07 확장하기

### 커스텀 도구

    from pydantic import BaseModel, Field
    from openharness.tools.base import BaseTool, ToolExecutionContext, ToolResult

    class MyToolInput(BaseModel):
        query: str = Field(description="Search query")

    class MyTool(BaseTool):
        name = "my_tool"
        description = "Does something useful"
        input_model = MyToolInput

        async def execute(self, args: MyToolInput, ctx: ToolExecutionContext) -> ToolResult:
            return ToolResult(output=f"Result for: {args.query}")

### 커스텀 스킬

`~/.openharness/skills/my-skill.md`:

    ---
    name: my-skill
    description: Expert guidance for my domain
    ---
    # My Skill
    ## When to use
    Use when the user asks about [domain].
    ## Workflow
    1. Step one
    2. Step two

### 커스텀 플러그인

`.openharness/plugins/my-plugin/.claude-plugin/plugin.json`:

    { "name": "my-plugin", "version": "1.0.0", "description": "My plugin" }

하위에 `commands/*.md`, `hooks/hooks.json`, `agents/*.md` 추가.

#### 공식 테스트 통과 플러그인 (12개)

| 플러그인          | 타입       | 기능                   |
|-------------------|------------|------------------------|
| commit-commands   | Commands   | Git commit/push/PR     |
| security-guidance | Hooks      | 파일 편집 시 보안 경고 |
| hookify           | Cmd+Agents | 커스텀 훅 생성         |
| feature-dev       | Commands   | 기능 개발 워크플로우   |
| code-review       | Agents     | 멀티에이전트 PR 리뷰   |
| pr-review-toolkit | Agents     | 특화 PR 리뷰           |

## 08 테스트 & 기여

| 스위트                | 수  | 상태                                       |
|-----------------------|-----|--------------------------------------------|
| Unit + Integration    | 114 | ✅ All passing                             |
| CLI Flags E2E         | 6   | ✅ Real model calls                        |
| Harness Features E2E  | 9   | ✅ Retry, skills, parallel, permissions    |
| React TUI E2E         | 3   | ✅ Welcome, conversation, status           |
| TUI Interactions E2E  | 4   | ✅ Commands, permissions, shortcuts        |
| Real Skills + Plugins | 12  | ✅ anthropics/skills + claude-code/plugins |

    uv run pytest -q                           # unit/integration
    python scripts/test_harness_features.py     # Harness E2E
    python scripts/test_real_skills_plugins.py  # Plugins E2E

### 기여 가이드

| 영역        | 예시                  |
|-------------|-----------------------|
| Tools       | 도메인 특화 도구      |
| Skills      | .md 도메인 지식       |
| Plugins     | commands+hooks+agents |
| Providers   | LLM 백엔드 확장       |
| Multi-Agent | 조율 프로토콜         |
| Testing     | E2E, 벤치마크         |
| Docs        | 가이드, 번역          |

    git clone https://github.com/HKUDS/OpenHarness.git
    cd OpenHarness && uv sync --extra dev && uv run pytest -q

**oh** — OpenHarness v0.1.2 · HKUDS · MIT License "The model is the agent. The code is the harness."
