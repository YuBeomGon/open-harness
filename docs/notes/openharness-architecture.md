핵심부 + 주변부 통합 문서

# *OpenHarness* Agent 아키텍처 완전 분석

에이전트의 3대 핵심(Loop · Tool · Context)과 4대 주변부(안전 · 확장 · Multi-Agent · 인터페이스)를 디렉토리, 다이어그램, 코드 레벨로 분석합니다.

## CORE Part 1 — 핵심부

에이전트를 에이전트로 만드는 3가지. 이것 없이는 그냥 챗봇.

## DIR 핵심부 디렉토리 구조

src/openharness/ ├─ engine/── 에이전트 루프 전체 │ ├─ query.py── **run_query()** — 핵심 while 루프 │ ├─ query_engine.py── QueryEngine 클래스 (세션 소유자) │ ├─ messages.py── ContentBlock 타입 시스템 (Text/ToolUse/ToolResult) │ ├─ stream_events.py── StreamEvent 6종 (TextDelta, TurnComplete, ...) │ └─ cost_tracker.py── UsageSnapshot 누적 ├─ tools/── 도구 실행 (43+ 도구) │ ├─ base.py── **BaseTool, ToolRegistry** │ ├─ bash_tool.py, file_read_tool.py, ...── 개별 도구 구현 │ └─ skill_tool.py── 스킬 로딩 도구 ├─ prompts/── 컨텍스트 조립 │ ├─ system_prompt.py── **\_BASE_SYSTEM_PROMPT** + Environment │ ├─ context.py── **build_runtime_system_prompt()** — 7-Layer 조립 │ ├─ claudemd.py── CLAUDE.md 탐색/주입 │ └─ environment.py── OS/shell/git 감지 ├─ services/compact/── 컨텍스트 윈도우 관리 │ └─ \_\_init\_\_.py── microcompact + full compact + auto_compact_if_needed ├─ memory/── 영속 메모리 │ ├─ paths.py, manager.py, scan.py, search.py │ └─ memdir.py── MEMORY.md 프롬프트 생성 └─ api/── LLM API 추상화    ├─ client.py── **SupportsStreamingMessages** Protocol + AnthropicApiClient    ├─ openai_client.py── OpenAI 호환    ├─ copilot_client.py, codex_client.py    └─ usage.py── UsageSnapshot(input_tokens, output_tokens)

## 1-1 The Loop — 에이전트의 존재 이유

이 `while` 루프 하나가 "챗봇"과 "에이전트"를 가릅니다. 모델이 `stop_reason="tool_use"`를 반환하는 한 루프를 계속합니다.

### 플로우차트

![](assets/openharness-architecture-01.svg)

Compact API 호출 분기 도구 실행 종료

### 코드 리뷰

#### QueryEngine — 세션 소유자 (query_engine.py)

    class QueryEngine:
        def __init__(self, *, api_client: SupportsStreamingMessages,
            tool_registry, permission_checker, model, system_prompt,
            max_tokens=4096, max_turns=8, hook_executor=None):
            self._messages: list[ConversationMessage] = []
            self._cost_tracker = CostTracker()

        async def submit_message(self, prompt):
            self._messages.append(ConversationMessage.from_user_text(prompt))
            context = QueryContext(...)  # 모든 의존성 번들
            async for event, usage in run_query(context, self._messages):
                if usage: self._cost_tracker.add(usage)
                yield event

#### run_query — 핵심 while 루프 (query.py)

    async def run_query(context, messages) -> AsyncIterator[...]:
        compact_state = AutoCompactState()
        turn_count = 0
        while context.max_turns is None or turn_count < context.max_turns:
            turn_count += 1
            # ① auto-compact
            messages, _ = await auto_compact_if_needed(messages, ...)
            # ② API 스트리밍
            async for event in api_client.stream_message(ApiMessageRequest(
                model=..., messages=messages, tools=tool_registry.to_api_schema())):
                if isinstance(event, ApiTextDeltaEvent): yield AssistantTextDelta(...)
                elif isinstance(event, ApiMessageCompleteEvent): final_message = ...
            messages.append(final_message)
            yield AssistantTurnComplete(...)
            # ③ 도구 호출 없으면 종료
            if not final_message.tool_uses: return
            # ④ 도구 실행 (단일→순차, 복수→asyncio.gather)
            tool_results = [await _execute_tool_call(...) for tc in tool_calls]
            messages.append(ConversationMessage(role="user", content=tool_results))

**핵심:** `messages` 리스트가 참조 전달되어 run_query 내부에서 직접 append. compact도 이 리스트를 in-place 변환합니다.

#### 병렬 실행

    if len(tool_calls) == 1:  # 순차: Started → await execute → Completed
        ...
    else:  # 병렬: Started×N 먼저 → asyncio.gather → Completed×N
        results = await asyncio.gather(*[_run(tc) for tc in tool_calls])

## 1-2 Tool Execution — 에이전트의 손

### 6단계 체인 다이어그램

![](assets/openharness-architecture-02.svg)

### 코드 리뷰

#### BaseTool — 모든 도구의 기반 (tools/base.py)

    class BaseTool(ABC):
        name: str; description: str
        input_model: type[BaseModel]  # Pydantic → JSON Schema 자동 생성
        async def execute(self, args, ctx: ToolExecutionContext) -> ToolResult: ...
        def is_read_only(self, args) -> bool: ...  # 권한 판단용
        def to_api_schema(self) -> dict: ...       # {"name", "description", "input_schema"}

    class ToolRegistry:
        def register(self, tool): self._tools[tool.name] = tool
        def to_api_schema(self) -> list[dict]:  # 전체 도구 스키마 → API 전달

#### ContentBlock 타입 시스템 (messages.py)

    ContentBlock = Annotated[TextBlock | ToolUseBlock | ToolResultBlock, Field(discriminator="type")]

    class ToolUseBlock:  # assistant가 생성
        type = "tool_use"; id: str; name: str; input: dict
    class ToolResultBlock:  # harness가 생성 → user 메시지에 담김
        type = "tool_result"; tool_use_id: str; content: str; is_error: bool

**Tool Use 판단:** 별도 classifier 모델 없음. 동일한 생성형 모델이 도구 스키마를 컨텍스트로 받고, "도구를 쓴다" = ToolUseBlock 토큰을 생성하는 것. Harness는 이 출력을 파싱해서 실행할 뿐.

## 1-3 Context Assembly — 에이전트의 기억

### 시스템 프롬프트 7-Layer 스택

![](assets/openharness-architecture-03.svg)

### Auto-Compact 2단계

![](assets/openharness-architecture-04.svg)

### 코드 리뷰

    def build_runtime_system_prompt(settings, *, cwd, latest_user_prompt):
        sections = [build_system_prompt(...)]  # L1: base+env
        sections += [session_settings]          # L2: effort, passes
        sections += [_build_skills_section()]   # L3: skill 목록
        sections += [load_claude_md_prompt()]   # L4: CLAUDE.md
        sections += [issue/pr context]          # L5
        sections += [load_memory_prompt()]      # L6: MEMORY.md index
        sections += [find_relevant_memories()]  # L7: 관련 메모리
        return "\n\n".join(sections)

    def estimate_tokens(text): return max(1, (len(text)+3)//4)  # 4자≈1토큰
    # × 4/3 패딩. 한글/CJK에서 과소추정 가능

## PERIPHERY Part 2 — 주변부

핵심부를 보호·확장·복제·표시하는 4가지 계층.

## DIR 주변부 디렉토리 구조

src/openharness/ ── 안전 계층 ── ├─ permissions/── modes.py (3모드), checker.py (PermissionChecker) ├─ hooks/── schemas.py (4타입), executor.py, loader.py, hot_reload.py ├─ sandbox/── adapter.py (OS-level sandboxing) ── 확장 계층 ── ├─ skills/── loader.py, registry.py, types.py, bundled/ ├─ plugins/── loader.py, schemas.py, types.py, installer.py ├─ mcp/── client.py (McpClientManager), config.py, types.py ── Multi-Agent ── ├─ coordinator/── TeamRegistry, agent_definitions.py ├─ swarm/── in_process.py, subprocess_backend.py, mailbox.py, types.py ├─ tasks/── manager.py (BackgroundTaskManager), local_agent_task.py ── 인터페이스 ── ├─ ui/── runtime.py (RuntimeBundle), backend_host.py, react_launcher.py ├─ commands/── registry.py (54 slash commands) ├─ config/── settings.py (Pydantic), paths.py └─ auth/── manager.py, external.py, flows.py

## 2-1 안전 계층 — 핵심부를 보호

루프(①)와 도구 실행(②) 사이에 끼어서 "이거 실행해도 되나?" 검증합니다.

### Permission 평가 체인

![](assets/openharness-architecture-05.svg)

### Hook 시스템

| Hook 타입 | 실행 방식                     | 판정                | block_on_failure |
|-----------|-------------------------------|---------------------|------------------|
| `command` | 셸 서브프로세스               | returncode == 0     | false            |
| `http`    | httpx POST                    | response.is_success | false            |
| `prompt`  | LLM → JSON {"ok": true/false} | ok 필드             | true             |
| `agent`   | LLM 심화 추론                 | 동일                | true             |

    async def execute(self, event: HookEvent, payload):
        for hook in self._registry.get(event):
            if _matches_hook(hook, payload):  # fnmatch matcher
                results.append(await self._run_xxx_hook(hook, ...))
        return AggregatedHookResult(results)  # .blocked = any(r.blocked)

**보안:** command hook의 `$ARGUMENTS`는 `shlex.quote()`로 shell-escape. `$(…)`/backtick injection 방어.

## 2-2 확장 계층 — 핵심부를 풍부하게

### Skills — On-Demand 지식

    def load_skill_registry(cwd, *, extra_skill_dirs, extra_plugin_roots):
        # 1순위: 번들 (내장) → 2순위: ~/.openharness/skills/*.md
        # 3순위: extra dirs → 4순위: 활성 플러그인의 스킬
        # → SkillRegistry(dict[name → SkillDefinition])

시스템 프롬프트에 스킬 목록이 포함되고, 모델이 `skill(name="commit")` 도구를 호출하면 해당 .md 파일의 content가 ToolResult로 반환됩니다.

### Plugins — claude-code 호환

    # 탐색 경로:
    # 1. ~/.openharness/plugins/{name}/(plugin.json 또는 .claude-plugin/plugin.json)
    # 2. {cwd}/.openharness/plugins/...
    # 3. extra_roots (ohmo 등)
    # 플러그인 구성: commands/*.md + hooks/hooks.json + agents/*.md + skills/*.md

### MCP — 외부 도구 서버

    class McpClientManager:
        # stdio MCP 서버에 연결, 도구/리소스 노출
        async def connect_all(self): ...     # 전체 서버 연결
        async def call_tool(self, server, name, args): ...
        async def read_resource(self, server, uri): ...
        # mcp_tool, list_mcp_resources, read_mcp_resource 도구와 연결

## 2-3 Multi-Agent — 루프를 복제

하나의 에이전트 루프가 또 다른 루프를 생성해서 병렬로 돌리는 구조입니다.

### Swarm 아키텍처 다이어그램

![](assets/openharness-architecture-06.svg)

### 코드 리뷰

#### TeammateExecutor Protocol (swarm/types.py)

    class TeammateExecutor(Protocol):
        type: BackendType  # "subprocess" | "in_process" | "tmux" | "iterm2"
        async def spawn(self, config: TeammateSpawnConfig) -> SpawnResult: ...
        async def send_message(self, agent_id, message): ...
        async def shutdown(self, agent_id, *, force=False): ...

#### AgentTool 스폰 흐름 (tools/agent_tool.py)

    async def execute(self, args, context):
        agent_def = get_agent_definition(args.subagent_type)  # 정의 조회
        executor = registry.get_executor("in_process")       # 백엔드 선택
        config = TeammateSpawnConfig(name=..., prompt=..., model=..., permissions=...)
        result = await executor.spawn(config)
        if args.team: get_team_registry().add_agent(args.team, result.task_id)

#### BackgroundTaskManager (tasks/manager.py)

    class BackgroundTaskManager:
        # 인메모리 task dict + asyncio.subprocess 관리
        async def create_shell_task(self, *, command, cwd): ...   # bash 백그라운드
        async def create_agent_task(self, *, prompt, cwd): ...   # 에이전트 서브프로세스
        # task_create/get/list/update/stop/output 도구와 연결

#### Mailbox (swarm/mailbox.py)

    # 파일 기반 async 메시지 큐
    # ~/.openharness/teams/{team}/agents/{agent_id}/inbox/{timestamp}_{id}.json
    # 원자적 쓰기: .tmp → os.rename (partial read 방지)
    # exclusive_file_lock으로 동시 접근 제어
    MessageType = Literal["user_message", "permission_request", "shutdown", ...]

## 2-4 인터페이스 계층 — 핵심부를 사용자에게 보여줌

### RuntimeBundle — 모든 것을 조립하는 지점

![](assets/openharness-architecture-07.svg)

### 코드 리뷰

#### RuntimeBundle (ui/runtime.py)

    @dataclass
    class RuntimeBundle:
        api_client: SupportsStreamingMessages
        cwd: str
        mcp_manager: McpClientManager
        tool_registry: ToolRegistry
        app_state: AppStateStore
        hook_executor: HookExecutor
        engine: QueryEngine           # ← 핵심부 루프
        commands: object              # 54 slash commands
        session_backend: SessionBackend

`build_runtime()`이 settings를 읽고, API 클라이언트·도구·권한·훅·MCP를 전부 생성해서 이 번들에 담습니다. TUI든 CLI든 ohmo든 이 번들을 받아서 동작합니다.

#### BackendHost — React TUI 프로토콜 (ui/backend_host.py)

    # Python 백엔드 ↔ React 프론트엔드: JSON-lines 프로토콜
    _PROTOCOL_PREFIX = "OHJSON:"
    # StreamEvent → OHJSON:{...}\n 형태로 stdout에 출력
    # React(Ink)가 이를 파싱하여 TUI 렌더링
    # 프론트엔드 → 백엔드: stdin으로 FrontendRequest JSON 전송

#### CommandRegistry (commands/registry.py)

    @dataclass
    class CommandResult:
        message: str | None = None
        should_exit: bool = False

    # 54개 slash commands: /help, /commit, /plan, /resume, /compact,
    # /memory, /permissions, /model, /clear, /config, /theme, ...
    # + 플러그인이 추가하는 commands/*.md

## ∴ 전체 구조 요약

![](assets/openharness-architecture-08.svg)

**oh** — OpenHarness Agent Architecture Complete Analysis 소스 기반: v0.1.2 (2026-04-06) · HKUDS · MIT License
