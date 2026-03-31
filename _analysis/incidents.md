# Type 2 Candidates — Incident Raw Material
*Analysis window: last 6 weeks (2026-02-18 – 2026-03-31). Older incidents not covered.*

---

## LangGraph Server — thread_id location (4 directives, distributed systems mechanics)

**Signals that generalize:** LangGraph SDK / LangGraph Server, config.configurable vs runtime.context split, distributed invocation context propagation
**Source directives:** D6
**Chronological findings:**
1. `ThreadDataMiddleware` only read `thread_id` from `runtime.context` — commit `e6c6770` (2026-03-22) [D6]
2. `MemoryMiddleware` had the same bug independently — commit `520c035` (2026-03-28) [D6]
3. `task_tool` had the same bug independently — commit `690d80f` (2026-03-28) [D6]
4. Lazy sandbox init had the same bug independently — commit `118485a` (2026-03-29) [D6]

**Candidate practice:** "When invoking a LangGraph agent via LangGraph Server (as opposed to direct Python invocation), invocation context (thread_id, user_id, etc.) lives in `config.configurable`, not `runtime.context`. Code that reads only one source silently fails in server mode. Always implement a two-source fallback: `runtime.context.get('thread_id') or config.configurable.get('thread_id')`."
**Status:** seed (1 team) — promote when second team confirms

---

## Gemini thinking + OpenAI-compatible gateway — thought_signature stripping (1 directive, third-party library)

**Signals that generalize:** LangChain `ChatOpenAI`, Gemini API, thinking content blocks, OpenAI-compatible proxy gateway, multi-turn conversations with tool calls
**Source directives:** D16
**Chronological findings:**
1. Standard `ChatOpenAI` silently drops `thought_signature` fields on Gemini thinking content blocks during message serialization. Gemini requires signatures echoed back verbatim; missing signatures cause HTTP 400 on the next turn — commit `a087fe7` (2026-03-26) [D16]

**Candidate practice:** "When using Gemini thinking models via an OpenAI-compatible gateway through LangChain's `ChatOpenAI`, the `thought_signature` field on thinking content blocks is silently dropped during serialization. This causes HTTP 400 on any multi-turn conversation that includes tool calls, because Gemini validates that signatures are preserved across turns. Use a patched adapter that re-injects signed thinking blocks from `additional_kwargs.thinking_blocks` or the content list before sending each request."
**Status:** seed (1 team) — promote when second team confirms

---

## Docker Compose — environment vs env_file precedence (1 directive, configuration/third-party tool)

**Signals that generalize:** Docker Compose, `environment` section, `env_file`, shell variable expansion, `${VAR:-default}` syntax
**Source directives:** D12
**Chronological findings:**
1. `LANGSMITH_TRACING=${LANGSMITH_TRACING:-false}` in `environment` section always resolved to `false` because Docker Compose evaluates `environment` via host shell before reading `env_file`. Users setting `LANGSMITH_TRACING=true` in `.env` saw no effect — commit `64e0f53` (2026-03-31) [D12]

**Candidate practice:** "In Docker Compose, `environment` entries take precedence over `env_file` entries for the same variable. The `${VAR:-default}` syntax is evaluated from the host shell environment at `docker compose up` time — before `env_file` is loaded. If you put `SOME_VAR=${SOME_VAR:-false}` in `environment` and expect users to override it via `.env`, the override will be silently ignored. Either remove the `environment` entry and rely solely on `env_file`, or document that the variable must be set in the shell environment (not `.env`) before running compose."
**Status:** seed (1 team) — promote when second team confirms

---

## Background task registry — unbounded growth + cleanup race condition (2 directives, resource management)

**Signals that generalize:** global in-memory task registry, background thread pool, polling loops, terminal state detection, race conditions
**Source directives:** D10, D11
**Chronological findings:**
1. Global `_background_tasks` dict accumulated completed subagent results indefinitely — commit `0409f8c` (2026-03-10) [D10]
2. Premature cleanup during polling-timeout caused `KeyError` when the background executor tried to update the still-running entry — commit `0409f8c` (2026-03-10) [D11]

**Candidate practice:** "A global dict used as a background task registry will grow unboundedly if completed entries are never removed. The safe cleanup pattern is: (1) only remove entries when status is terminal (COMPLETED/FAILED/TIMED_OUT) or `completed_at` is set; (2) never remove on polling-timeout because the background thread may still be running and will KeyError when it tries to update the removed entry. The terminal-state guard is the critical invariant."
**Status:** seed (1 team) — promote when second team confirms
