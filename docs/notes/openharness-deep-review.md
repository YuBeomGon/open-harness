Source-Level Deep Dive

# *OpenHarness* 코드 레벨 리뷰

에이전트 파이프라인, 컨텍스트 관리, 권한/훅 시스템, Multi-Agent 아키텍처를 소스 코드 수준에서 분석합니다.

## 01 Agent Loop Pipeline — 전체 실행 흐름

진입점submit_message사용자 텍스트 → messages에 추가 → 컨텍스트QueryContextapi, tools, perms, hooks 번들 → 매 턴 시작auto_compact토큰 체크 → micro/full → API 호출stream_message스트리밍 + retry → 분기stop_reasontool_use? → 계속 end_turn → 종료 → 도구 실행_execute_tool_callPreHook → Perm → Exec → PostHook ↩ **핵심 설계:** 모델이 `stop_reason="tool_use"`를 반환하는 한 루프를 계속합니다. `max_turns` (기본 200, QueryEngine 레벨에서는 8)으로 무한 루프를 방지합니다.

### QueryEngine — 세션 소유자

`engine/query_engine.py`는 대화 히스토리와 비용 추적을 소유하며, `run_query`에 위임합니다.

    class QueryEngine:
        """대화 히스토리 + CostTracker 소유. run_query에 실행 위임."""

        def __init__(self, *,
            api_client: SupportsStreamingMessages,  # Protocol 기반 — DI 가능
            tool_registry: ToolRegistry,
            permission_checker: PermissionChecker,
            model: str,  system_prompt: str,
            max_tokens: int = 4096,
            max_turns: int | None = 8,  # ← QueryContext에서는 200
            hook_executor: HookExecutor | None = None,
        ):
            self._messages: list[ConversationMessage] = []
            self._cost_tracker = CostTracker()

        async def submit_message(self, prompt: str) -> AsyncIterator[StreamEvent]:
            self._messages.append(ConversationMessage.from_user_text(prompt))
            context = QueryContext(...)  # 모든 의존성을 번들로 전달
            async for event, usage in run_query(context, self._messages):
                if usage: self._cost_tracker.add(usage)
                yield event

설계 포인트 `messages` 리스트는 `run_query`에 참조로 전달됩니다. `run_query` 내부에서 직접 append하므로, QueryEngine이 별도로 동기화할 필요가 없습니다. Auto-compact도 이 리스트를 직접 변환합니다.

#### continue_pending — 중단된 도구 루프 재개

    def has_pending_continuation(self) -> bool:
        # 마지막 메시지가 user role + ToolResultBlock이고
        # 그 직전 assistant 메시지에 tool_uses가 있으면 True

    async def continue_pending(self, *, max_turns=None):
        # 새 user 메시지 없이 run_query 재진입
        # 세션 복원(resume) 시 사용

### run_query — 핵심 에이전트 루프

`engine/query.py` — 전체 에이전트의 실질적인 심장부입니다. async generator로 구현되어 모든 이벤트를 yield합니다.

    async def run_query(context: QueryContext, messages: list) -> AsyncIterator[...]:
        compact_state = AutoCompactState()
        turn_count = 0

        while context.max_turns is None or turn_count < context.max_turns:
            turn_count += 1

            # ┌─── PHASE 1: Auto-Compact ───────────────────────┐
            messages, was_compacted = await auto_compact_if_needed(
                messages, api_client=..., model=..., state=compact_state)
            # └──────────────────────────────────────────────────┘

            # ┌─── PHASE 2: API Stream ─────────────────────────┐
            try:
                async for event in context.api_client.stream_message(
                    ApiMessageRequest(
                        model=..., messages=messages,
                        system_prompt=..., max_tokens=...,
                        tools=context.tool_registry.to_api_schema(),  # 전체 도구 스키마
                    )
                ):
                    if isinstance(event, ApiTextDeltaEvent):
                        yield AssistantTextDelta(...), None  # 스트리밍 텍스트
                    elif isinstance(event, ApiRetryEvent):
                        yield StatusEvent(...), None          # 재시도 알림
                    elif isinstance(event, ApiMessageCompleteEvent):
                        final_message = event.message
                        usage = event.usage
            except Exception as exc:
                # 네트워크/API 오류 → ErrorEvent yield 후 return
            # └──────────────────────────────────────────────────┘

            messages.append(final_message)
            yield AssistantTurnComplete(message=final_message, usage=usage), usage

            # ┌─── PHASE 3: 분기 ────────────────────────────────┐
            if not final_message.tool_uses:
                return  # 모델이 도구 없이 응답 → 루프 종료
            # └──────────────────────────────────────────────────┘

            # ┌─── PHASE 4: Tool Execution ─────────────────────┐
            # (단일이면 순차, 복수면 asyncio.gather로 병렬)
            # └──────────────────────────────────────────────────┘

            messages.append(ConversationMessage(role="user", content=tool_results))
            # ↩ while 루프 상단으로 → 다음 턴

        raise MaxTurnsExceeded(context.max_turns)

### \_execute_tool_call — 도구 실행의 전체 체인

하나의 도구 호출이 거치는 전체 파이프라인입니다:

    async def _execute_tool_call(context, tool_name, tool_use_id, tool_input):

        # ① PreToolUse Hook
        if context.hook_executor:
            pre_hooks = await hook_executor.execute(HookEvent.PRE_TOOL_USE, {
                "tool_name": tool_name,
                "tool_input": tool_input,
            })
            if pre_hooks.blocked:
                return ToolResultBlock(..., is_error=True)  # 훅이 차단

        # ② Tool Registry Lookup
        tool = context.tool_registry.get(tool_name)
        if tool is None:
            return ToolResultBlock(content=f"Unknown tool: {tool_name}", is_error=True)

        # ③ Pydantic Input Validation
        try:
            parsed_input = tool.input_model.model_validate(tool_input)
        except Exception:
            return ToolResultBlock(..., is_error=True)  # 입력 검증 실패

        # ④ Permission Evaluation
        _file_path = _resolve_permission_file_path(cwd, tool_input, parsed_input)
        _command = _extract_permission_command(tool_input, parsed_input)
        decision = context.permission_checker.evaluate(
            tool_name, is_read_only=tool.is_read_only(parsed_input),
            file_path=_file_path, command=_command)

        if not decision.allowed:
            if decision.requires_confirmation and permission_prompt:
                confirmed = await permission_prompt(tool_name, decision.reason)
                if not confirmed: return ...  # 사용자 거부
            else: return ...  # 무조건 거부

        # ⑤ Tool Execution (with timing)
        t0 = time.monotonic()
        result = await tool.execute(parsed_input, ToolExecutionContext(
            cwd=context.cwd,
            metadata={"tool_registry": ..., "ask_user_prompt": ...}))
        elapsed = time.monotonic() - t0

        # ⑥ PostToolUse Hook
        if context.hook_executor:
            await hook_executor.execute(HookEvent.POST_TOOL_USE, {
                "tool_name": ..., "tool_output": ..., "tool_is_error": ...})

6단계 체인 PreHook → Registry Lookup → Pydantic Validate → Permission Check → Execute → PostHook. 어느 단계에서든 실패하면 `ToolResultBlock(is_error=True)`를 반환하여 모델에게 오류를 알립니다.

### 병렬 vs 순차 도구 실행

    # 단일 도구: 즉시 이벤트 스트리밍
    if len(tool_calls) == 1:
        tc = tool_calls[0]
        yield ToolExecutionStarted(...), None
        result = await _execute_tool_call(context, tc.name, tc.id, tc.input)
        yield ToolExecutionCompleted(...), None
        tool_results = [result]

    # 복수 도구: asyncio.gather로 동시 실행
    else:
        for tc in tool_calls:
            yield ToolExecutionStarted(...), None  # 시작 이벤트 먼저 전부 발행

        results = await asyncio.gather(*[_run(tc) for tc in tool_calls])

        for tc, result in zip(tool_calls, results):
            yield ToolExecutionCompleted(...), None  # 완료 이벤트 일괄 발행

**설계 의도:** 모델이 독립적인 도구 여러 개를 한 번에 호출하면 (예: 파일 3개 동시 읽기) 대기 시간을 줄입니다. 단일 호출 시에는 Started → Completed 사이에 실시간 피드백이 가능합니다.

## 02 메시지 모델 — ContentBlock 타입 시스템

    # Pydantic discriminated union — type 필드로 자동 분기
    ContentBlock = Annotated[
        TextBlock | ToolUseBlock | ToolResultBlock,
        Field(discriminator="type")
    ]

    class TextBlock(BaseModel):
        type: Literal["text"] = "text"
        text: str

    class ToolUseBlock(BaseModel):
        type: Literal["tool_use"] = "tool_use"
        id: str = Field(default_factory=lambda: f"toolu_{uuid4().hex}")
        name: str
        input: dict[str, Any]

    class ToolResultBlock(BaseModel):
        type: Literal["tool_result"] = "tool_result"
        tool_use_id: str       # ToolUseBlock.id와 매칭
        content: str
        is_error: bool = False

    class ConversationMessage(BaseModel):
        role: Literal["user", "assistant"]
        content: list[ContentBlock]

        # 편의 프로퍼티
        @property
        def tool_uses(self) -> list[ToolUseBlock]: ...  # assistant msg에서 도구 호출 추출
        @property
        def text(self) -> str: ...                     # 텍스트 블록만 연결

패턴 assistant 메시지 = TextBlock + ToolUseBlock 혼합. 도구 결과는 role="user" 메시지에 ToolResultBlock으로 담깁니다. 이 구조가 Anthropic Messages API의 turn-based 모델과 정확히 대응합니다.

### 직렬화 — API 와이어 포맷 변환

    def serialize_content_block(block: ContentBlock) -> dict:
        # TextBlock → {"type": "text", "text": "..."}
        # ToolUseBlock → {"type": "tool_use", "id": ..., "name": ..., "input": ...}
        # ToolResultBlock → {"type": "tool_result", "tool_use_id": ..., "content": ...}

    def assistant_message_from_api(raw_message) -> ConversationMessage:
        # Anthropic SDK 객체 → 내부 ConversationMessage로 변환
        # getattr 기반으로 SDK 버전 변경에 견고하게 대응

**OpenAI 호환 변환**은 `openai_client.py`의 `_convert_messages_to_openai`에서 처리됩니다. system prompt → role="system" 메시지, ToolResultBlock → role="tool" 메시지, ToolUseBlock → tool_calls 필드로 변환합니다.

## 03 API 추상화 계층

### Protocol 패턴 — 테스트 가능한 추상화

    class SupportsStreamingMessages(Protocol):
        """구조적 서브타이핑 — 상속 없이 duck typing으로 호환"""
        async def stream_message(self, request: ApiMessageRequest) -> AsyncIterator[ApiStreamEvent]: ...

이 Protocol을 만족하는 4개 구현체:

| 클라이언트           | 파일              | SDK            | 특이사항                                               |
|----------------------|-------------------|----------------|--------------------------------------------------------|
| `AnthropicApiClient` | client.py         | AsyncAnthropic | OAuth 지원, Claude Subscription bridge                 |
| `OpenAIApiClient`    | openai_client.py  | AsyncOpenAI    | 메시지/도구 포맷 자동 변환, max_completion_tokens 분기 |
| `CopilotApiClient`   | copilot_client.py | httpx          | GitHub Device Flow OAuth                               |
| `CodexApiClient`     | codex_client.py   | bridge         | ~/.codex/auth.json 크레덴셜                            |

### AnthropicApiClient — Retry 상세

    MAX_RETRIES = 3; BASE_DELAY = 1.0; MAX_DELAY = 30.0
    RETRYABLE_STATUS_CODES = {429, 500, 502, 503, 529}

    async def stream_message(self, request):
        for attempt in range(MAX_RETRIES + 1):
            try:
                self._refresh_client_auth()    # OAuth 토큰 갱신 체크
                async for event in self._stream_once(request):
                    yield event
                return
            except OpenHarnessApiError: raise  # 인증 오류는 재시도 안 함
            except Exception as exc:
                if not _is_retryable(exc): raise
                delay = _get_retry_delay(attempt, exc)  # 지수 백오프 + jitter
                yield ApiRetryEvent(...)  # UI에 재시도 알림
                await asyncio.sleep(delay)

Retry 전략 `_get_retry_delay`: ① Retry-After 헤더 존재 시 해당 값 사용 → ② 없으면 `min(BASE_DELAY × 2^attempt, 30s) + random(0, delay×0.25)`. Jitter로 thundering herd 방지.

### OpenAI 호환 — 포맷 변환의 핵심

    def _token_limit_param_for_model(model, max_tokens):
        # GPT-5, o1, o3, o4 계열 → max_completion_tokens 사용
        # 그 외 → max_tokens 사용

    def _convert_tools_to_openai(tools):
        # Anthropic: {"name", "description", "input_schema"}
        # → OpenAI: {"type": "function", "function": {"name", "description", "parameters"}}

    def _convert_messages_to_openai(messages, system_prompt):
        # system_prompt → {"role": "system", "content": ...}  (Anthropic은 별도 파라미터)
        # ToolUseBlock → assistant msg의 tool_calls 필드
        # ToolResultBlock → {"role": "tool", "tool_call_id": ..., "content": ...}

## 04 권한 시스템 — 평가 체인

    def evaluate(self, tool_name, *, is_read_only, file_path=None, command=None):

        # ① SENSITIVE_PATH_PATTERNS — 무조건 거부, 설정 불가
        if file_path:
            for pattern in SENSITIVE_PATH_PATTERNS:
                if fnmatch.fnmatch(file_path, pattern):
                    return PermissionDecision(allowed=False)

        # ② denied_tools 명시적 차단
        if tool_name in self._settings.denied_tools: return Decision(allowed=False)

        # ③ allowed_tools 명시적 허용
        if tool_name in self._settings.allowed_tools: return Decision(allowed=True)

        # ④ path_rules (fnmatch glob) — deny 규칙만 체크
        for rule in self._path_rules:
            if fnmatch.fnmatch(file_path, rule.pattern) and not rule.allow:
                return Decision(allowed=False)

        # ⑤ denied_commands (fnmatch)
        for pattern in denied_commands:
            if fnmatch.fnmatch(command, pattern): return Decision(allowed=False)

        # ⑥ PermissionMode 분기
        if mode == FULL_AUTO:  return Decision(allowed=True)
        if is_read_only:       return Decision(allowed=True)
        if mode == PLAN:       return Decision(allowed=False)    # 쓰기 완전 차단
        # DEFAULT → requires_confirmation=True (사용자에게 물어봄)

**Defence-in-depth:** `SENSITIVE_PATH_PATTERNS`는 하드코딩이며 사용자 설정이나 PermissionMode로 오버라이드할 수 없습니다. 프롬프트 인젝션으로 `.ssh/`, `.aws/credentials` 등에 접근하는 것을 원천 차단합니다.

### 경로 해석 로직

    def _resolve_permission_file_path(cwd, raw_input, parsed_input):
        # raw_input dict에서 "file_path" 또는 "path" 키를 찾음
        # → expanduser → 상대경로면 cwd 기준으로 resolve
        # → 절대경로 문자열 반환

    def _extract_permission_command(raw_input, parsed_input):
        # "command" 키에서 셸 커맨드 문자열 추출

설계 Pydantic 검증 **전**에 raw_input과 parsed_input 양쪽에서 경로/커맨드를 추출합니다. 도구마다 file_path/path 키 이름이 다를 수 있어서 양쪽 모두 확인합니다.

## 05 Hook 시스템

### 4가지 Hook 타입의 실행 메커니즘

| 타입      | 실행 방식                                   | 결과 판단                 | block_on_failure 기본값 |
|-----------|---------------------------------------------|---------------------------|-------------------------|
| `command` | 셸 서브프로세스 (`create_shell_subprocess`) | returncode == 0 → success | false                   |
| `http`    | httpx POST (JSON payload)                   | response.is_success       | false                   |
| `prompt`  | LLM 호출 → JSON 응답 파싱                   | `{"ok": true/false}`      | true                    |
| `agent`   | LLM 호출 (더 심화된 추론)                   | 동일                      | true                    |

### HookExecutor — 실행 흐름 상세

    async def execute(self, event: HookEvent, payload):
        results = []
        for hook in self._registry.get(event):
            if not _matches_hook(hook, payload): continue
            # hook.type에 따라 적절한 실행기 호출
            results.append(await self._run_xxx_hook(hook, event, payload))
        return AggregatedHookResult(results=results)
        # .blocked → any(r.blocked for r in results)
        # .reason → 첫 번째 blocking reason

#### Matcher 패턴

    def _matches_hook(hook, payload):
        matcher = getattr(hook, "matcher", None)
        if not matcher: return True  # matcher 없으면 모든 도구에 적용
        subject = payload.get("tool_name") or payload.get("prompt") or ...
        return fnmatch.fnmatch(subject, matcher)  # e.g. "bash" or "edit_*"

#### Prompt/Agent Hook의 LLM 판정

    # 시스템 프롬프트: "Return strict JSON: {"ok": true} or {"ok": false, "reason": "..."}"
    # agent_mode=True이면 "Be more thorough" 추가
    # max_tokens=512로 짧게 제한

    def _parse_hook_json(text):
        # JSON 파싱 시도 → 실패하면 "ok"/"true"/"yes" 문자열 체크
        # 그것도 아니면 {"ok": False, "reason": text}

### 보안: \$ARGUMENTS Shell Escape

    def _inject_arguments(template, payload, *, shell_escape=False):
        serialized = json.dumps(payload, ensure_ascii=True)
        if shell_escape:
            serialized = shlex.quote(serialized)  # ← command hook에만 적용
        return template.replace("$ARGUMENTS", serialized)

**CHANGELOG에서 수정된 취약점:** 이전에는 `$ARGUMENTS`에 `$(…)`이나 backtick이 포함되면 shell injection이 가능했습니다. `shlex.quote()`로 감싸서 모든 메타문자를 이스케이프합니다. HTTP/Prompt hook에는 shell이 아니므로 escape 없이 raw JSON을 전달합니다.

## 06 컨텍스트 관리 — Auto-Compact 파이프라인

Claude Code의 compaction 시스템을 충실히 재구현한 `services/compact/__init__.py`입니다.

매 턴should_autocompact토큰 추정 ≥ threshold? → 1차microcompact오래된 tool result 클리어 비용: 0 (LLM 호출 없음) → 충분?재측정threshold 미만이면 종료 → 2차compact_conversationLLM 요약 호출 비용: API 1회 → 결과summary + newerolder → 요약 메시지로 교체

#### Threshold 계산

    def get_autocompact_threshold(model):
        context_window = get_context_window(model)  # 기본 200,000
        reserved = min(MAX_OUTPUT_TOKENS_FOR_SUMMARY, 20_000)
        effective = context_window - reserved       # 180,000
        return effective - AUTOCOMPACT_BUFFER_TOKENS  # 180,000 - 13,000 = 167,000

### Microcompact — 무료 토큰 절약

    COMPACTABLE_TOOLS = frozenset({
        "read_file", "bash", "grep", "glob",
        "web_search", "web_fetch", "edit_file", "write_file"})

    def microcompact_messages(messages, *, keep_recent=5):
        # 1. assistant 메시지에서 COMPACTABLE_TOOLS의 tool_use ID 수집
        # 2. 최근 keep_recent개 제외, 나머지의 tool_result content를
        #    "[Old tool result content cleared]"로 교체
        # 3. messages를 in-place로 변환 (효율성)
        return messages, tokens_saved

핵심 read_file 같은 큰 출력을 가진 도구의 오래된 결과를 지웁니다. 모델은 이미 그 내용을 처리했으므로, 최근 5개만 보존하면 충분합니다.

### Full Compact — LLM 요약

    async def compact_conversation(messages, *, api_client, model, preserve_recent=6):
        # Step 1: microcompact 먼저 실행
        messages, _ = microcompact_messages(messages)

        # Step 2: 분할
        older = messages[:-preserve_recent]  # → 요약 대상
        newer = messages[-preserve_recent:]  # → 그대로 보존

        # Step 3: LLM에 compact prompt 전송 (도구 없이)
        # "Respond with TEXT ONLY. Do NOT call any tools."
        # → <analysis> 블록 + <summary> 블록 요청
        summary_text = await call_llm(older + [compact_prompt])

        # Step 4: 결과 조립
        # format_compact_summary: <analysis> 제거, <summary> 추출
        # build_compact_summary_message: 세션 연속성 안내 텍스트 추가
        return [summary_msg, *newer]

#### Compact Prompt의 요약 섹션 (9개)

1.  Primary Request and Intent
2.  Key Technical Concepts
3.  Files and Code Sections (경로 + 라인 번호)
4.  Errors and Fixes
5.  Problem Solving (성공/실패한 접근)
6.  All User Messages (원문 보존)
7.  Pending Tasks (미완료 작업)
8.  Current Work (마지막 작업 상태)
9.  Optional Next Step

**AutoCompactState:** 연속 실패 `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES=3`회 도달 시 auto-compact를 비활성화합니다. 무한 재시도로 토큰을 낭비하는 것을 방지합니다.

### 토큰 추정

    def estimate_tokens(text: str) -> int:
        return max(1, (len(text) + 3) // 4)  # 4문자 ≈ 1토큰 (거친 휴리스틱)

    def estimate_message_tokens(messages):
        total = sum(블록별 estimate_tokens)
        return int(total * TOKEN_ESTIMATION_PADDING)  # × 4/3 보수적 패딩

트레이드오프 tiktoken 같은 정확한 토크나이저 대신 단순 문자 수 기반 추정을 사용합니다. 속도 우선 + 4/3 패딩으로 안전 마진 확보. 한글/CJK 문자에서는 실제보다 과소추정될 수 있습니다.

## 07 시스템 프롬프트 조립

### build_runtime_system_prompt — 전체 조립 과정

    def build_runtime_system_prompt(settings, *, cwd, latest_user_prompt, ...):
        sections = []

        # ① Base prompt + Environment (OS, shell, cwd, git branch, date)
        sections.append(build_system_prompt(custom_prompt=settings.system_prompt, cwd=cwd))

        # ② Fast mode / Reasoning settings (effort, passes)
        if settings.fast_mode: sections.append("# Session Mode\nFast mode...")
        sections.append(f"# Reasoning Settings\n- Effort: {settings.effort}\n...")

        # ③ Available Skills 목록
        skills_section = _build_skills_section(cwd, ...)
        if skills_section: sections.append(skills_section)

        # ④ CLAUDE.md (프로젝트 인스트럭션)
        claude_md = load_claude_md_prompt(cwd)
        if claude_md: sections.append(claude_md)

        # ⑤ Issue Context / PR Comments (있으면)
        for title, path in [("Issue Context", ...), ("PR Comments", ...)]:
            if path.exists(): sections.append(f"# {title}\n```md\n{content[:12000]}\n```")

        # ⑥ MEMORY.md 엔트리포인트
        if settings.memory.enabled:
            memory_section = load_memory_prompt(cwd, max_entrypoint_lines=200)
            if memory_section: sections.append(memory_section)

        # ⑦ Relevant Memories (사용자 입력 기반 검색)
            if latest_user_prompt:
                relevant = find_relevant_memories(latest_user_prompt, cwd, max_results=5)
                # 각 메모리 파일 content[:8000] 주입

        return "\n\n".join(sections)

7개 레이어 Base + Env → Settings → Skills → CLAUDE.md → Issue/PR → Memory Index → Relevant Memories. 각 레이어가 독립적으로 존재/부재할 수 있으며, 모두 `\n\n`으로 결합됩니다.

### CLAUDE.md 탐색 — 프로젝트 인스트럭션

    def discover_claude_md_files(cwd):
        current = Path(cwd).resolve()
        for directory in [current, *current.parents]:
            # 우선순위:
            # 1. {dir}/CLAUDE.md
            # 2. {dir}/.claude/CLAUDE.md
            # 3. {dir}/.claude/rules/*.md (정렬됨)
        return results  # seen set으로 중복 제거

    def load_claude_md_prompt(cwd, *, max_chars_per_file=12000):
        # 발견된 모든 파일을 "# Project Instructions" 섹션으로 주입
        # 파일당 12,000자 초과 시 truncate

### 메모리 주입 — 사용자 입력 기반 검색

    # MEMORY.md 엔트리포인트 (항상 주입)
    "# Memory\n- Persistent memory directory: {path}\n## MEMORY.md\n```md\n{content}\n```"

    # Relevant Memories (사용자 최신 프롬프트와 관련된 파일만)
    relevant = find_relevant_memories(latest_user_prompt, cwd, max_results=5)
    # → "# Relevant Memories\n## {filename}\n```md\n{content[:8000]}\n```"

## 08 Skills & Plugin 시스템

### 스킬 로딩 체인

    def load_skill_registry(cwd, *, extra_skill_dirs, extra_plugin_roots):
        registry = SkillRegistry()     # dict[name → SkillDefinition]

        # 1순위: 번들 스킬 (내장)
        for skill in get_bundled_skills(): registry.register(skill)

        # 2순위: 사용자 스킬 (~/.openharness/skills/*.md)
        for skill in load_user_skills(): registry.register(skill)

        # 3순위: extra_skill_dirs (ohmo 등에서 추가)
        for skill in load_skills_from_dirs(extra_skill_dirs): registry.register(skill)

        # 4순위: 활성 플러그인의 스킬
        for plugin in load_plugins(...):
            if plugin.enabled:
                for skill in plugin.skills: registry.register(skill)

        return registry

#### SkillDefinition — 데이터 모델

    @dataclass(frozen=True)
    class SkillDefinition:
        name: str           # YAML frontmatter의 name
        description: str    # frontmatter의 description
        content: str        # 전체 .md 파일 내용 (모델에 주입됨)
        source: str         # "bundled" / "user" / "plugin"
        path: str | None

#### SkillTool — 모델이 스킬을 요청하는 방법

    class SkillTool(BaseTool):
        name = "skill"
        # 시스템 프롬프트에 스킬 목록이 포함됨
        # 모델이 "skill(name="commit")"을 호출하면
        # → skill.content를 ToolResult로 반환
        # → 모델이 그 내용을 읽고 작업 수행

### 플러그인 호환성 — claude-code 포맷

디렉토리 구조: `.openharness/plugins/{name}/.claude-plugin/plugin.json`

하위에 `commands/*.md` + `hooks/hooks.json` + `agents/*.md`. Hook은 `load_hook_registry`에서 settings의 hooks와 플러그인의 hooks를 합쳐서 HookRegistry에 등록합니다.

## 09 Memory 시스템

### 경로 설계

    def get_project_memory_dir(cwd):
        path = Path(cwd).resolve()
        digest = sha1(str(path).encode()).hexdigest()[:12]
        # ~/.openharness/data/memory/OpenHarness-a1b2c3d4e5f6/
        return get_data_dir() / "memory" / f"{path.name}-{digest}"

설계 프로젝트명 + SHA1 12자로 충돌 방지. 같은 이름의 다른 경로에 있는 프로젝트도 구분됩니다.

#### CRUD API (manager.py)

    def add_memory_entry(cwd, title, content) -> Path:
        slug = re.sub(r"[^a-zA-Z0-9]+", "_", title.lower()).strip("_")
        path = memory_dir / f"{slug}.md"
        path.write_text(content)
        # MEMORY.md 인덱스에 "- [title](slug.md)" 추가
        return path

    def remove_memory_entry(cwd, name) -> bool:
        # 파일 삭제 + MEMORY.md에서 해당 라인 제거

### 검색 스코어링

    def find_relevant_memories(query, cwd, *, max_results=5):
        tokens = _tokenize(query)  # Han character aware tokenizer

        for header in scan_memory_files(cwd, max_files=100):
            meta = f"{header.title} {header.description}".lower()
            body = header.body_preview.lower()

            meta_hits = sum(1 for t in tokens if t in meta)
            body_hits = sum(1 for t in tokens if t in body)
            score = meta_hits * 2.0 + body_hits  # frontmatter 가중치 2배

        # score 내림차순 → modified_at 내림차순 정렬

**다국어 지원:** `_tokenize`에서 Han character (CJK) 토크나이저를 사용하여 한글/중국어/일본어 쿼리도 올바르게 분할합니다. (CHANGELOG에서 수정됨)

## 10 Multi-Agent / Swarm 아키텍처

### TeammateExecutor Protocol — 실행 백엔드 추상화

    @runtime_checkable
    class TeammateExecutor(Protocol):
        type: BackendType  # "subprocess" | "in_process" | "tmux" | "iterm2"

        def is_available(self) -> bool: ...
        async def spawn(self, config: TeammateSpawnConfig) -> SpawnResult: ...
        async def send_message(self, agent_id, message: TeammateMessage): ...
        async def shutdown(self, agent_id, *, force=False) -> bool: ...

#### TeammateSpawnConfig — 에이전트 스폰 설정

    @dataclass
    class TeammateSpawnConfig:
        name: str              # "researcher", "tester" 등
        team: str              # 소속 팀
        prompt: str            # 초기 프롬프트
        cwd: str               # 작업 디렉토리
        parent_session_id: str # 트랜스크립트 연결용
        model: str | None      # 모델 오버라이드
        system_prompt: str | None
        permissions: list[str]
        plan_mode_required: bool = False
        worktree_path: str | None  # git worktree 격리
        subscriptions: list[str]   # 이벤트 토픽 구독

### AgentTool → 스폰 흐름

    async def execute(self, args, context):
        # 1. subagent_type이 있으면 agent_definitions에서 정의 조회
        agent_def = get_agent_definition(args.subagent_type)

        # 2. BackendRegistry에서 executor 선택
        #    in_process → subprocess → 기본 순서로 fallback
        executor = registry.get_executor("in_process")

        # 3. TeammateSpawnConfig 생성 → executor.spawn(config)
        config = TeammateSpawnConfig(
            name=agent_name, team=team, prompt=args.prompt,
            model=args.model or agent_def.model,
            system_prompt=agent_def.system_prompt,
            permissions=agent_def.permissions)

        result = await executor.spawn(config)

        # 4. team이 지정되면 TeamRegistry에 등록
        if args.team:
            get_team_registry().add_agent(args.team, result.task_id)

        return ToolResult(f"Spawned agent {result.agent_id} (task={result.task_id})")

#### Pane-based 백엔드 (tmux / iTerm2)

`PaneBackend` Protocol은 시각적 멀티에이전트 경험을 제공합니다:

- `create_teammate_pane_in_swarm_view` — 새 터미널 패인 생성
- `send_command_to_pane` — 패인에 커맨드 전송
- `set_pane_border_color / title` — 에이전트별 시각적 구분
- `hide_pane / show_pane` — 패인 숨기기/보이기
- `rebalance_panes` — 레이아웃 재조정 (leader 패인 고려)

**전체 Swarm 구성:** swarm/ 디렉토리에 in_process.py (async 동일 프로세스 실행), subprocess_backend.py (별도 프로세스), worktree.py (git worktree 격리), team_lifecycle.py, permission_sync.py, mailbox.py (에이전트 간 메시지 전달), lockfile.py가 포함되어 있습니다. **oh** — OpenHarness Deep Code Review 소스 기반: v0.1.2 (2026-04-06) · HKUDS · MIT
