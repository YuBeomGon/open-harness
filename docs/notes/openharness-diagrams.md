Companion Diagrams for Deep Code Review

# Agent *3대 핵심* 동작 다이어그램

OpenHarness 에이전트의 핵심은 3가지뿐입니다. 나머지는 전부 이것들을 보호하거나 확장하는 것.

① The Loop ② Tool Execution ③ Context Assembly

## 01 The Loop — 에이전트의 존재 이유

이 `while` 루프 하나가 "챗봇"과 "에이전트"를 가르는 경계입니다. 모델이 **"도구를 더 써야 한다"**고 말하는 한 루프를 계속합니다.

### 전체 에이전트 루프 플로우차트

![](assets/openharness-diagrams-01.svg)

Compact 체크 API 호출 분기 판단 도구 실행 종료 루프 흐름 **핵심 포인트:** 모든 것은 이 루프 안에서 일어납니다. 모델은 "다음에 뭘 할지" 결정하고, 코드(harness)는 "그것을 어떻게 실행할지" 처리합니다. `stop_reason`이 `"tool_use"`가 아닌 순간 — 모델이 "할 일 끝났어"라고 판단한 순간 — 루프가 끝납니다.

### 한 턴(Turn) 안에서 일어나는 일

![](assets/openharness-diagrams-02.svg)

### 병렬 실행 vs 순차 실행

![](assets/openharness-diagrams-03.svg)

# 단일 → 순차: Started yield → await execute → Completed yield
    # 복수 → 병렬: Started×N → asyncio.gather → Completed×N
    if len(tool_calls) == 1:  # 순차
        yield ToolExecutionStarted(...) → await execute → yield Completed(...)
    else:                       # 병렬
        [yield Started for tc] → await asyncio.gather(*) → [yield Completed]

## 02 Tool Execution — 에이전트의 손

모델이 "이 도구 써줘"라고 요청하면, 실제로 실행되기까지 **6단계 체인**을 거칩니다. 어느 단계에서든 실패하면 에러를 모델에게 알려주고 끝납니다.

### \_execute_tool_call 6단계 체인

![](assets/openharness-diagrams-04.svg)

**어느 단계에서든 실패 →** `ToolResultBlock(is_error=True)`를 반환합니다. 모델은 이 에러를 읽고 "다른 접근을 시도"하거나 "사용자에게 알림"합니다. 에러도 대화의 일부입니다.

### Permission Evaluation 체인 상세

![](assets/openharness-diagrams-05.svg)

## 03 Context Assembly — 에이전트의 기억

모델이 "똑똑하게" 행동하는 건 코드가 매 턴마다 조립해서 보내는 **컨텍스트** 덕분입니다. 두 파트로 나뉩니다: 시스템 프롬프트 조립과 대화 히스토리 압축.

### 시스템 프롬프트 7-Layer 조립

![](assets/openharness-diagrams-06.svg)

**왜 이렇게 많은 레이어?** 모델은 컨텍스트에 있는 것만 "알고" 있습니다. 프로젝트 규칙, 이전 세션 기억, 현재 이슈 — 이것들을 매번 조립해서 넣어줘야 모델이 "이 프로젝트를 아는 것처럼" 행동합니다.

### Auto-Compact — 컨텍스트 윈도우 관리

대화가 길어지면 컨텍스트 윈도우(200K 토큰)를 초과합니다. 매 턴 시작 시 체크하고, 2단계로 압축합니다.

![](assets/openharness-diagrams-07.svg)

**왜 2단계?** Microcompact는 무료입니다 (LLM 호출 없이 오래된 도구 출력만 지움). 이것만으로 충분하면 비싼 Full Compact를 건너뜁니다. "가능한 한 싸게, 필요하면 비싸게" 전략입니다.

### 모델이 실제로 보는 것

![](assets/openharness-diagrams-08.svg)

**모든 것은 텍스트입니다.** 모델에게 "도구"란 결국 JSON 스키마가 시스템 프롬프트에 포함되고, 도구 결과가 메시지로 돌아오는 것입니다. 모델은 이 텍스트 컨텍스트만 보고 다음 행동을 결정합니다. Harness가 하는 일은 이 컨텍스트를 잘 조립하는 것이 전부입니다.

## ∴ 정리 — 에이전트 = 이 3가지의 조합

![](assets/openharness-diagrams-09.svg)

**oh** — OpenHarness Agent Core Diagrams · Companion to Deep Code Review 코드 리뷰와 나란히 보세요 → `openharness-deep-review.md`
