# Cluster E: Safety and Security

## SECTION 9: GUARDRAILS AND AI SAFETY

**Guardrail inventory:**

| **Guardrail type** | **Applied at** | **Implemented?** | **Evidence** |
|---|---|---|---|
| Prompt injection detection | Input | No | No implementation |
| Input content filtering | Input | No | No implementation |
| Output toxicity / bias filtering | Output | No | No implementation |
| PII scrubbing from outputs | Output | No | No implementation |
| Behavioral constraints via system prompt | All turns | **Yes** | `agents/lead_agent/prompt.py:162-348` |
| Tool use restrictions | Mid-task | **Yes (optional)** | `guardrails/middleware.py`, `guardrails/builtin.py` |
| External moderation API | Input / Output | No | No implementation |
| Hallucination or grounding check | Output | No | No implementation |
| Scope containment | Input | No (prompt-only) | `agents/lead_agent/prompt.py:179-245` |
| Model fallback on safety trigger | Mid-task | No | No implementation |
| Loop detection / forced stop | Mid-task | **Yes** | `agents/middlewares/loop_detection_middleware.py` |
| XSS prevention (active content artifacts) | Output | **Yes** | `app/gateway/routers/artifacts.py:16-20` |
| Host bash restriction | Tool execution | **Yes** | `sandbox/security.py:35-45` |

**Tool use restrictions (GuardrailMiddleware):**
- Optional feature — enabled when `guardrails.enabled: true` in `config.yaml`
- Evaluates every tool call against configured `GuardrailProvider` before execution
- Denied calls return error `ToolMessage`; agent continues with alternative approach
- `fail_closed: true` default — provider errors block the call
- Built-in: `AllowlistProvider` — simple allowed/denied tool name sets — evidence: `guardrails/builtin.py`
- OAP-compatible providers: external policy enforcement (e.g. `aport-agent-guardrails`)
- Evidence: `guardrails/middleware.py:20-99`

**Behavioral constraints (system prompt):**
- Explicit `<clarification_system>` section: 5 mandatory clarification scenarios (missing info, ambiguous requirement, approach choice, risk confirmation, suggestion) with `REQUIRED ACTION` rules
- `<critical_reminders>`: clarification first rule, output directory enforcement, language consistency
- Working directory constraints: agent restricted to `/mnt/user-data/uploads`, `/mnt/user-data/workspace`, `/mnt/user-data/outputs`
- Citations required for web search results
- Agent explicitly blocked from starting work before clarifying ambiguous requests
- Evidence: `agents/lead_agent/prompt.py:178-245`

**Loop detection:**
- `LoopDetectionMiddleware` tracks tool call hash per thread in sliding window of 20
- Warn threshold (3 identical calls): injects `HumanMessage` warning ("you are repeating yourself")
- Hard limit (5 identical calls): strips tool_calls from AIMessage, forces text output
- LRU eviction at 100 threads; thread-safe with `threading.Lock`
- Evidence: `agents/middlewares/loop_detection_middleware.py:36-213`

**XSS prevention:**
- Artifact endpoint forces `Content-Disposition: attachment` for `text/html`, `application/xhtml+xml`, `image/svg+xml` — prevents browser from executing active content in application origin
- Evidence: `app/gateway/routers/artifacts.py:16-20`, `172-173`

**Host bash restriction:**
- `LocalSandboxProvider` + `allow_host_bash: false` (default): bash tool excluded from tool list
- Explicit error message if bash subagent attempted with local sandbox
- Evidence: `sandbox/security.py:35-45`, `tools/tools.py:59-60`

**Human-in-the-loop:**
- `ask_clarification` tool: model calls → `ClarificationMiddleware` intercepts → `Command(goto=END)` → LangGraph interrupts execution → waits for user input
- Evidence: `agents/middlewares/clarification_middleware.py`
- Required for: missing info, ambiguous requirements, risky operations, approach choices, suggestions (per system prompt rules)
- No escalation path when guardrails fire (agent receives error ToolMessage, adapts on its own)
- No timeout for human response — LangGraph interruption holds state indefinitely
- Correction integration: user message resumes the same thread, agent continues from interrupt
- UX: chat interface in frontend; IM channels receive and respond inline

---

## SECTION 10: AUTHENTICATION, AUTHORIZATION, AND SECURITY

**Input validation and injection prevention:**
- FastAPI Pydantic models validate API request schemas at gateway boundary — framework-level
- Path traversal: `resolve_thread_virtual_path()` resolves virtual paths to actual filesystem paths; artifacts endpoint returns 403 on traversal detection
- No explicit SQL injection protection (no SQL in agent path; checkpointer uses LangGraph's own SQLite layer)
- No prompt injection detection or sanitization before user input reaches LLM

**Data protection:**
- Log sanitization: No PII scrubbing in logs — messages logged at debug/info level may contain user content
- Error detail sanitization: `ToolErrorHandlingMiddleware` truncates error messages to 500 chars — evidence: `middlewares/tool_error_handling_middleware.py:26-27`
- Artifact active content forced to download to prevent XSS
- ACP workspace accessible at `/mnt/acp-workspace` read-only — evidence: `CLAUDE.md`

**Rate limiting:**
- No implementation at API level, tool level, or LLM call level
- Subagent concurrency limited to 3 concurrent executions (`SubagentLimitMiddleware`, `MAX_CONCURRENT_SUBAGENTS = 3`) — evidence: `subagents/executor.py:456`
- Memory fact limit (100) caps memory growth — evidence: `agents/memory/updater.py:404-410`

**Authentication:**
- Frontend: `better-auth` session management (requires `BETTER_AUTH_SECRET` env var) — evidence: `docker/docker-compose.yaml:53`
- Backend Gateway API: No authentication on endpoints — relies on nginx/network-level access control
- LangGraph Server: No authentication beyond what langgraph-sdk provides
