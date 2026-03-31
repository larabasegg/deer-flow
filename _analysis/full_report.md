# Agent Codebase Analysis Report
**Codebase**: /Users/lara.baseggio/Documents/deer-flow
**Date**: 2026-03-31

# Cluster A: Core Architecture

## SECTION 0: AGENT METADATA

### Project Identity

| **Attribute** | **Answer** | **Source (file:line)** |
|---|---|---|
| Project name | DeerFlow | `backend/pyproject.toml:2` |
| Stated purpose | "LangGraph-based AI agent system with sandbox execution capabilities" | `backend/pyproject.toml:4` |
| Use case domain | General-purpose super-agent: web research, code execution, multi-agent delegation, persistent memory | `backend/CLAUDE.md`, `README.md` |
| Described as "agent"? | Yes — "DeerFlow is a LangGraph-based AI super agent system" | `backend/CLAUDE.md:1` |

### Authorship and Provenance

| **Attribute** | **Answer** | **Source** |
|---|---|---|
| Author(s) or organization | Community open source (ByteDance origin per README demo images; no explicit author field) | `pyproject.toml`, `README.md` |
| License | Not present in repo root (LICENSE file exists but not read for this analysis) | `LICENSE` |
| Version | 0.1.0 | `backend/pyproject.toml:3` |

---

## SECTION 1: SDK, LLM STACK AND MODEL CONFIGURATION

**Which LLMs are used:**

| **Model slot** | **Provider SDK** | **Class path** | **Notes** |
|---|---|---|---|
| Lead agent (default) | Any — config-driven | Resolved via `model_config.use` string | Could be OpenAI, Anthropic, Gemini, DeepSeek, etc. |
| Summarization model | Same factory | `create_chat_model(name=config.model_name, thinking_enabled=False)` | Falls back to default model if `model_name: null` |
| Title generation model | Same factory | `create_chat_model(name=config.model_name)` | Falls back to default |
| Memory updater model | Same factory | `create_chat_model(name=config.model_name)` | Falls back to default |
| Subagent model | Same factory | `create_chat_model(name=model_name, thinking_enabled=False)` | Inherits or overrides parent model via `config.model == "inherit"` |

- Evidence: `backend/packages/harness/deerflow/models/factory.py:11-95`
- Model configuration examples in `config.example.yaml:36-233`

**Model selection — dynamic, config-driven:**
- `make_lead_agent()` resolves model at runtime: request `model_name` → agent config model → global default (first in `models[]`)
- Evidence: `backend/packages/harness/deerflow/agents/lead_agent/agent.py:26-38`
- No A/B testing or ML-based routing — selection is deterministic based on config and request parameter

**Model routing by task complexity:**
- Not implemented — same model handles all tasks. Only exception: summarization, title, and memory can be assigned a cheaper model via `model_name: null` fallback.

**Fallback model:**
- Partial — if requested model not found in config, falls back to first model in `models[]` list with a warning log
- Evidence: `backend/packages/harness/deerflow/agents/lead_agent/agent.py:26-38`
- No chain of fallbacks; one fallback level only

**Thinking mode:**
- `thinking_enabled` runtime flag passed through `config.configurable`
- Per-model `supports_thinking` + `when_thinking_enabled` overrides control actual behavior
- If model doesn't support thinking: warning logged, `thinking_enabled` forced to `False`
- Evidence: `backend/packages/harness/deerflow/models/factory.py:43-56`

**Token usage tracking:**
- Implemented via `TokenUsageMiddleware` — enabled when `config.token_usage.enabled: true`
- Logs at INFO level per model call; no budget ceiling or automatic truncation/switching
- Evidence: `backend/packages/harness/deerflow/agents/middlewares/token_usage_middleware.py`, `config.example.yaml:28-30`

**Tool binding:**
- Standard LangChain tool binding via `create_agent(model=..., tools=[...])`
- No explicit `strict=True` or `parallel_tool_calls=False` observed in factory
- `SubagentLimitMiddleware` enforces max concurrent `task` calls at the application layer (not API level)
- Evidence: `backend/packages/harness/deerflow/agents/lead_agent/agent.py:332-348`

---

## SECTION 2A: AGENT ARCHITECTURE — INVENTORY

**1A — Execution units:**

| **Name** | **Has tool selection discretion?** | **Can loop?** | **What it does** |
|---|---|---|---|
| `lead_agent` | Yes (10+ tools) | Yes (LangGraph react loop) | Main reasoning agent — handles user requests, delegates, uses tools |
| `general-purpose` subagent | Yes (all tools except `task`) | Yes | Handles delegated research/analysis sub-tasks in background |
| `bash` subagent | Yes (bash-group tools) | Yes | Handles delegated command execution sub-tasks |
| `SummarizationMiddleware` (constrained LLM) | No — forced to summarize | No — single call | Summarizes conversation history when token limit approaches |
| `TitleMiddleware` (constrained LLM) | No — forced to generate title | No — single call | Auto-generates thread title after first exchange |
| `MemoryMiddleware` updater (constrained LLM) | No — forced memory extraction | No — single call | Extracts structured facts from conversation |

**1B — Control flow logic:**

| **Name / location** | **Mechanism** | **What it checks** | **Where it routes to** |
|---|---|---|---|
| `ClarificationMiddleware` (`agent.py:270`) | Event-driven — intercepts `ask_clarification` tool call | Tool call name == `ask_clarification` | `Command(goto=END)` — interrupts execution |
| `SubagentLimitMiddleware` (`agent.py:256-259`) | Rule-based | Count of `task` tool calls > max_concurrent | Truncates excess task calls from model response |
| `LoopDetectionMiddleware` | Rule-based | Repetitive tool call pattern detection | Breaks loop — prevents infinite cycles |
| `DeferredToolFilterMiddleware` | Rule-based | Tool has deferred flag | Hides tool schema from model binding when tool_search enabled |
| `_resolve_model_name()` (`agent.py:26-38`) | Rule-based | Requested model in config? | Routes to requested model or falls back to default |

**1C — Tools (lead_agent):**

| **Tool name** | **Bound to** | **Required or optional** | **Description** |
|---|---|---|---|
| `bash` | lead_agent, bash subagent | Optional (model chooses) | Execute shell commands in sandbox |
| `ls` | lead_agent, both subagents | Optional | List directory contents |
| `read_file` | lead_agent, both subagents | Optional | Read file with optional line range |
| `write_file` | lead_agent, both subagents | Optional | Write/append file contents |
| `str_replace` | lead_agent, both subagents | Optional | Substring replacement in files |
| `web_search` | lead_agent, general-purpose | Optional | Web search (DDG/Tavily/InfoQuest) |
| `web_fetch` | lead_agent, general-purpose | Optional | Fetch web page content |
| `image_search` | lead_agent, general-purpose | Optional | Image search |
| `present_files` | lead_agent only | Optional | Make output files visible to user |
| `ask_clarification` | lead_agent only | Optional | Interrupt for user clarification |
| `view_image` | lead_agent (vision models only) | Optional | Read image as base64 |
| `task` | lead_agent (if subagent_enabled) | Optional | Delegate work to a subagent |
| `write_todos` | lead_agent (if plan_mode) | Optional | Manage todo task list |
| `tool_search` | lead_agent (if tool_search enabled) | Optional | Load deferred MCP tool schemas |
| `invoke_acp_agent` | lead_agent (if ACP configured) | Optional | Invoke external ACP agent |
| MCP tools | lead_agent (if MCP enabled) | Optional | Any tool exposed by enabled MCP servers |

---

## SECTION 2B: AGENT ARCHITECTURE — CLASSIFICATION

**2A — Execution unit classification:**

- **`lead_agent`**: Agent — LLM calls + tool discretion (10+ tools) + looping (LangGraph react cycle)
- **`general-purpose` subagent**: Agent — LLM calls + tool discretion + looping
- **`bash` subagent**: Agent — LLM calls + limited tool discretion (bash-group tools) + looping
- **`SummarizationMiddleware`**: Constrained LLM step — LLM call forced to summarize; no tool discretion, no loop. Needed because summarization requires semantic compression, not rule-based truncation
- **`TitleMiddleware`**: Constrained LLM step — forced to generate short title; no tools, no loop. Needed because title generation requires language understanding
- **`MemoryMiddleware` updater**: Constrained LLM step — forced to extract structured facts; no tools, no loop. Needed because fact extraction requires semantic understanding

**2B — System-level classification:**

| **Level** | **Criteria** | **Applies?** |
|---|---|---|
| Direct model call | Zero agents | No |
| Single agent with tools | Exactly one agent | No |
| Multi-agent orchestration | Two or more agents | **Yes** |

**2C — Multi-agent patterns:**

| **Pattern** | **Present?** | **Evidence** |
|---|---|---|
| Sequential / Pipeline | No | Subagents run concurrently, not sequenced |
| Concurrent / Parallel | **Yes** | `task` tool launches multiple subagents simultaneously via `execute_async()` |
| Supervisor / Subagents | **Yes** | `lead_agent` (supervisor) delegates via `task` tool to `general-purpose`/`bash` agents |
| Handoff / Transfer | No | No explicit agent handoff mechanism |
| Router / Dispatch | No | Lead agent uses LLM discretion, not a dedicated router |
| Group chat / Roundtable | No | No shared conversation thread between agents |

**2D — Identity cards:**

**Agent: lead_agent**

| **Attribute** | **Value** |
|---|---|
| Name / ID | `lead_agent` (or custom `agent_name` from config) |
| Description | Open-source super agent handling user requests end-to-end |
| Capabilities | All available tools: sandbox (bash/ls/read/write/str_replace), web (search/fetch/image), builtin (present_files/ask_clarification/view_image), subagent (task), MCP tools, todo (write_todos), ACP (invoke_acp_agent) |
| Model | Config-driven; first model in `models[]` by default |
| System prompt | Role definition → soul (optional) → memory injection → thinking style → clarification system → skills → deferred tools → subagent instructions → working directory → response style → citations → reminders |
| Registration | `langgraph.json` → `"lead_agent": "deerflow.agents:make_lead_agent"` |

**Agent: general-purpose subagent**

| **Attribute** | **Value** |
|---|---|
| Name / ID | `general-purpose` |
| Description | All-purpose sub-task executor for research, analysis, file operations |
| Capabilities | All lead_agent tools minus `task` (no recursive subagents) |
| Model | Inherits parent or per-config override; `thinking_enabled=False` |
| System prompt | Defined in `deerflow/subagents/builtins/__init__.py` |
| Registration | Hardcoded in `SubagentRegistry` |

**Agent: bash subagent**

| **Attribute** | **Value** |
|---|---|
| Name / ID | `bash` |
| Description | Specialist for shell command execution tasks |
| Capabilities | bash-group tools only |
| Model | Inherits parent or per-config override; `thinking_enabled=False` |
| System prompt | Defined in `deerflow/subagents/builtins/__init__.py` |
| Registration | Hardcoded in `SubagentRegistry` |

**Constrained LLM steps:**

| **Name** | **Purpose** | **What it's forced to do** | **Why LLM needed** |
|---|---|---|---|
| SummarizationMiddleware | Context reduction | Produce a compact summary of older conversation history | Semantic compression requires language understanding |
| TitleMiddleware | Thread titling | Generate short title (≤6 words, ≤60 chars) for the conversation | Title quality requires understanding conversation context |
| MemoryMiddleware updater | Persistent memory | Extract structured facts, categories, confidence from conversation | Fact extraction and deduplication require semantic understanding |

---

## SECTION 2C: AGENT ARCHITECTURE — COMMUNICATION, ROUTING, AND PROMPTS

**Communication and state:**
- All agents share `ThreadState` — sandbox state, thread_data paths, message history passed through
- Subagents inherit sandbox and thread_data from parent via `SubagentExecutor._build_initial_state()` — evidence: `subagents/executor.py:182-201`
- Subagent results returned as tool call results in lead_agent's message history
- Error handling: exceptions in subagent caught internally → `SubagentStatus.FAILED` with error message returned to lead_agent — evidence: `subagents/executor.py:343-389`
- No formal inter-agent protocol (A2A etc.) — coordination is via: (1) `task` tool call → `SubagentExecutor.execute_async()` → background thread → poll → result as ToolMessage

**Routing:**
- No dedicated router — lead_agent uses LLM discretion for all routing decisions
- Clarification interrupt: `ClarificationMiddleware` intercepts `ask_clarification` → `Command(goto=END)` — rule-based
- Subagent selection: lead_agent LLM chooses `subagent_type` parameter in `task` tool call — LLM-based
- Thread-level routing: IM channels route to `lead_agent` via langgraph-sdk HTTP — rule-based
- Custom agents: `agent_name` configurable routes to custom SOUL.md + tool groups — config-based

**System prompts:**
- Structure: XML-tagged sections — `<role>`, `<soul>`, `<memory>`, `<thinking_style>`, `<clarification_system>`, `<skill_system>`, `<available-deferred-tools>`, `<subagent_system>`, `<working_directory>`, `<response_style>`, `<citations>`, `<critical_reminders>`
- Storage: hardcoded Python template string in `agents/lead_agent/prompt.py`
- Dynamic injection at runtime: memory content, skills list, subagent instructions (conditional), ACP section (conditional), deferred tools list
- No versioning system — prompt is in source code; rollback via git
- Evidence: `backend/packages/harness/deerflow/agents/lead_agent/prompt.py:162-348`

---

## SECTION 2D: AGENT ARCHITECTURE — MCP

- **MCP used**: Yes — `langchain-mcp-adapters` `MultiServerMCPClient` for multi-server tool integration
- Evidence: `backend/packages/harness/deerflow/mcp/client.py:1`

**Client-server setup:**
- Client: `MultiServerMCPClient` in `deerflow/mcp/tools.py` — lazy initialized, mtime-cached
- Servers: user-configured in `extensions_config.json` → `mcpServers` map
- Transports: `stdio` (subprocess command), `sse`, `http`
- OAuth support: token endpoint flows (`client_credentials`, `refresh_token`) for `http`/`sse` — evidence: `deerflow/mcp/oauth.py`
- Runtime updates: Gateway API saves config → LangGraph detects mtime change → reinitializes

**MCP vs direct API:**
- MCP chosen as extensibility mechanism — allows users to add any MCP-compatible server without code changes
- Built-in tools (web_search, web_fetch, sandbox) are direct integrations — not MCP
- MCP used for third-party tools to avoid hardcoding integrations
- Evidence: `config.example.yaml:352-354` (tool_search), `extensions_config.example.json`
# Cluster B: Execution, State, Memory

## SECTION 3: CONTROL FLOW AND PLANNING

**Branching:**
- Standard LangGraph react loop — model output routes back to tools node or to END based on whether tool calls are present
- `ClarificationMiddleware` injects `Command(goto=END)` when `ask_clarification` tool is called — evidence: `agents/middlewares/clarification_middleware.py`
- No custom conditional edges beyond standard react; LangGraph handles the loop internally via `create_agent()`
- Evidence: `agents/lead_agent/agent.py:332-348`

**Parallel/concurrent execution:**
- Tool calls can run concurrently within a single turn if model emits multiple tool calls (LangChain default behavior)
- Subagent tasks run in parallel via `SubagentExecutor.execute_async()` + `ThreadPoolExecutor` — evidence: `subagents/executor.py:391-453`
- Dual thread pool: `_scheduler_pool` (3 workers) schedules tasks; `_execution_pool` (3 workers) executes them
- `SubagentLimitMiddleware` enforces max concurrent `task` calls per response (default: 3)

**Replanning / re-routing:**
- No explicit replanning mechanism — lead_agent may issue new tool calls after receiving subagent results, but this is ad-hoc LLM behavior
- `LoopDetectionMiddleware` breaks infinite loops by detecting repetitive tool call patterns
- Summarization triggered when token count / message count / fraction threshold is exceeded

**Prompt chaining:**
- Present but implicit: memory update flow is a chain: MemoryMiddleware queues messages → queue debounce → `MemoryUpdater.update_memory()` → LLM call → structured JSON → `_apply_updates()` — evidence: `agents/memory/updater.py:269-333`
- Title generation: after first exchange, `TitleMiddleware` calls LLM → short title string → stored in `ThreadState.title`
- All chains are sequential single-call; no multi-hop LLM chains with intermediate validation

**Gates / checkpoints between chain steps:**
- Memory update: JSON parse check after LLM response; logs warning and returns `False` on `JSONDecodeError` — evidence: `updater.py:328-329`
- Title generation: character/word limit enforced by prompt instructions (not programmatic validation)
- No formal schema validation between chain steps

**Static vs dynamic chains:**
- All chains are static — memory update, title generation, and summarization follow fixed paths
- No early short-circuit logic; all execute unconditionally when triggered

---

## SECTION 4: STATE MANAGEMENT

**State structure design choices:**

`ThreadState` extends LangGraph `AgentState` with domain-specific fields:

| **Field** | **Type** | **Update mechanism** | **Notes** |
|---|---|---|---|
| `messages` (from AgentState) | `list` | LangGraph append reducer | Standard message accumulation |
| `sandbox` | `SandboxState | None` | Overwrite | Holds `sandbox_id` for current sandbox |
| `thread_data` | `ThreadDataState | None` | Overwrite | Holds workspace/uploads/outputs paths |
| `title` | `str | None` | Overwrite | Thread title (set by TitleMiddleware once) |
| `artifacts` | `Annotated[list[str], merge_artifacts]` | Custom reducer: deduplicate-merge | Artifact paths accumulate without duplication |
| `todos` | `list | None` | Overwrite | Todo items from plan mode |
| `uploaded_files` | `list[dict] | None` | Overwrite | Uploaded file list, refreshed each turn |
| `viewed_images` | `Annotated[dict, merge_viewed_images]` | Custom reducer: merge / empty-clears | Image base64 cache; `{}` signals clear |

- Evidence: `agents/thread_state.py:48-55`

**Custom reducers:**
- `merge_artifacts()`: deduplicates while preserving insertion order via `dict.fromkeys()` — evidence: `thread_state.py:21-28`
- `merge_viewed_images()`: merges dicts; empty dict `{}` explicitly clears all — evidence: `thread_state.py:31-45`

**State persistence:**
- LangGraph Server manages its own persistence independently (built-in)
- `DeerFlowClient` uses configurable checkpointer:
  - `memory` → `InMemorySaver` (process lifetime only)
  - `sqlite` → `AsyncSqliteSaver` / `SqliteSaver` (file-based, survives restarts)
  - `postgres` → `AsyncPostgresSaver` (multi-process production)
- Evidence: `agents/checkpointer/async_provider.py:41-77`
- Default in `config.example.yaml`: `type: sqlite, connection_string: checkpoints.db`

**TTL and key structure:**
- Key: `thread_id` (from `config.configurable.thread_id`)
- No TTL — LangGraph checkpointer stores indefinitely; cleanup is manual (Gateway thread delete endpoint)
- Evidence: `thread_data_middleware.py:78-84`, `app/gateway/routers/threads.py`

**State scopes:**
- Per-turn: `viewed_images` cleared after each ViewImageMiddleware processing cycle
- Per-thread: all `ThreadState` fields scoped to `thread_id`
- Global: memory file shared across all threads (unless per-agent memory used)
- No per-user state scope (user = thread for now)

**Concurrency risks:**
- Memory file writes use temp-file-then-rename atomic pattern — evidence: `agents/memory/storage.py:142-146`
- `FileMemoryStorage` uses in-process cache keyed by `agent_name`; no distributed locking (single-process safe)
- Subagent background tasks use `threading.Lock` for `_background_tasks` dict — evidence: `subagents/executor.py:69-75`
- No locking for `ThreadState` itself — LangGraph framework handles state isolation per thread

---

## SECTION 5: MEMORY MANAGEMENT

**Memory types:**

- **Semantic memory (facts)**: Implemented and active — discrete facts with `id`, `content`, `category` (preference/knowledge/context/behavior/goal), `confidence` (0–1), `createdAt`, `source` — evidence: `agents/memory/storage.py:18-34`
- **Episodic memory**: Partial — `history` sections (`recentMonths`, `earlierContext`, `longTermBackground`) summarize conversation history, but no few-shot examples stored; evidence: `updater.py:364-371`
- **Procedural memory**: No implementation — agent behavior defined by static system prompt; skills are static files, not dynamically updated

**Short-term memory (context window):**
- Kept: full message history (user + AI + tool calls + tool results) up to the summarization trigger
- Hard cap: no explicit message count cap; enforced indirectly by `SummarizationMiddleware` when token/message/fraction threshold is hit
- Ordering: preserved — older messages summarized, recent messages kept verbatim
- Tool calls: included in message history like other messages; no special handling
- System prompt: never truncated; injected on every turn
- User input truncation: No implementation — user messages added to state without truncation
- LLM-based summarization: `SummarizationMiddleware` (via `langchain.agents.middleware`); triggered when configured thresholds exceeded (e.g., 15564 tokens)
- Evidence: `config.example.yaml:494-526`, `agents/lead_agent/agent.py:41-80`

**Long-term memory (persistent):**

| **Type** | **What stored** | **Retrieval** | **TTL** | **Key** |
|---|---|---|---|---|
| File-based JSON (`memory.json`) | User context summaries, history summaries, facts (≤100 by default) | Load on prompt injection; mtime-cached | No TTL | `None` = global, or `agent_name` for per-agent |
| LangGraph checkpointer (SQLite/Postgres) | Full `ThreadState` per turn | LangGraph resumes from latest checkpoint | No TTL | `thread_id` |
| Thread data dirs (filesystem) | Per-thread workspace/uploads/outputs files | Direct file access via sandbox tools | No TTL; deleted when thread deleted | `thread_id` subdirectory |

**Cross-session memory:**
- `memory.json` is cross-session and cross-thread — all conversations share one global memory file (unless per-agent isolation configured)
- `thread_id` checkpointer is per-session (conversation-scoped)

**Shared vs isolated memory:**
- Memory shared across agents by default; per-agent isolation available by passing `agent_name` to memory functions
- Evidence: `agents/memory/storage.py:76-86` (per-agent file path), `updater.py:415-427`

**Memory injection:**
- Injected via `_get_memory_context()` in `apply_prompt_template()` as `<memory>` XML block in system prompt
- Top 15 facts + context summaries, limited to `max_injection_tokens: 2000` tokens
- Evidence: `agents/lead_agent/prompt.py:351-379`

**User profile / preference store:**
- Implemented in `memory.json` under `user.workContext`, `user.personalContext`, `user.topOfMind`
- Built incrementally via LLM extraction after each conversation
- Profile summarization: active — `MemoryUpdater._apply_updates()` updates summary strings when `shouldUpdate: true`
- Evidence: `agents/memory/updater.py:354-362`

**Memory disable:**
- Disabled per-request: No — memory injection is controlled by `config.yaml` flags only (`memory.enabled`, `memory.injection_enabled`)
- Master switches in `MemoryConfig` allow disabling globally; no per-request override mechanism
- Evidence: `config.example.yaml:540-549`, `agents/lead_agent/prompt.py:360-363`

**Upload mention filtering:**
- Special logic strips file-upload sentences from memory before saving (uploaded files are session-scoped, would cause hallucinations in future sessions)
- Evidence: `agents/memory/updater.py:209-240`
# Cluster C: Tools and Retrieval

## SECTION 6: TOOLS AND TOOL CALLING

**Tool binding and scoping:**

- Tools assembled per-agent-creation call in `get_available_tools()` — evidence: `deerflow/tools/tools.py:35-132`
- Lead agent and subagents have different tool sets:
  - Lead agent: all groups + builtin (present_files, ask_clarification, view_image, task, write_todos, tool_search, invoke_acp_agent) + MCP
  - General-purpose subagent: all groups + builtin minus `task` (no recursive delegation)
  - Bash subagent: bash-group tools only
  - Evidence: `subagents/executor.py:78-105`

- Tool set is dynamic per-request:
  - `groups` filter: custom agents restrict to their configured tool groups
  - `subagent_enabled`: `task` tool included only when enabled in runtime config
  - `model_name`: `view_image_tool` included only when model has `supports_vision: true`
  - `allow_host_bash: false` (default): bash tool excluded when LocalSandboxProvider active
  - MCP tools: loaded at call time from `extensions_config.json`, mtime-cached
  - Evidence: `tools.py:56-132`

- Force-required tools: No tool is always-forced; all tools are optional for model discretion
- `ask_clarification` is always in lead agent's tool list but model is not forced to call it

- Parallel tool calls: No explicit `parallel_tool_calls=False` — LangChain default allows them; `SubagentLimitMiddleware` limits parallel `task` calls at application level

**Tool execution:**

- Standard LangGraph tool node execution; no special post-tool router beyond react loop
- After tool executes → `ToolMessage` appended to state → model re-invoked (react loop)
- `ClarificationMiddleware` intercepts `ask_clarification` before execution → `Command(goto=END)` — evidence: `middlewares/clarification_middleware.py`
- `GuardrailMiddleware.wrap_tool_call()` / `awrap_tool_call()`: pre-execution authorization; returns error `ToolMessage` on deny — evidence: `guardrails/middleware.py`
- `SandboxAuditMiddleware`: wraps all tool calls for audit logging
- No direct-execution bypass path — all tool calls go through middleware chain
- Retry logic: No automatic retry on tool failure; `ToolErrorHandlingMiddleware` catches exceptions and returns error `ToolMessage` with message "Continue with available context, or choose an alternative tool" — evidence: `middlewares/tool_error_handling_middleware.py:22-35`
- Error handling: exception detail truncated to 500 chars in error ToolMessage — evidence: `tool_error_handling_middleware.py:26-27`

---

## SECTION 7: RAG AND RETRIEVAL

**RAG implementation:**
- No implementation — no vector database, embedding model, or RAG pipeline
- Retrieval is entirely tool-based: agent chooses and calls web search / web fetch tools at runtime

**Retrieval sources:**

| **Source** | **Tool** | **Provider** |
|---|---|---|
| Web search | `web_search_tool` | DuckDuckGo (default), Tavily (optional), InfoQuest (optional) |
| Web page content | `web_fetch_tool` | Jina AI reader (default), Firecrawl (optional), InfoQuest (optional) |
| Image search | `image_search_tool` | DuckDuckGo, InfoQuest |
| File content | `read_file_tool` | Local sandbox filesystem |
| MCP-connected sources | MCP tools | Any user-configured MCP server |
| ACP agent output | `invoke_acp_agent` | External ACP agent subprocess |

**Chunking:** No implementation — web content fetched as-is; no document chunking

**Embedding model:** No implementation

**Vector database:** No implementation

**Retrieval method:**
- Web search: keyword-based web queries via search APIs (DDG, Tavily, InfoQuest)
- No vector similarity, no hybrid BM25

**LLM reranking:** No implementation — agent decides which results to use based on snippets

**Integration:** Agentic RAG pattern — agent calls retrieval tools iteratively as needed during reasoning; no fixed retrieval stage

**Retrieval caching:**
- Jina fetch: `timeout: 10` enforced; no result caching
- MCP tools: cached at initialization level (not result-level)
- No result-level cache

**Multiple retrieval strategies:** Configured by user in `config.yaml` + `extensions_config.json` — agent always calls the configured tool; no dynamic strategy selection

**No-results behavior:** No special handling — agent receives empty or minimal response from tool and must reason about next steps; tool errors surface as `ToolMessage` with error status via `ToolErrorHandlingMiddleware`
# Cluster D: Data and Adaptation

## SECTION 8: LEARNING AND ADAPTATION

**Does the system learn from past interactions?**

Partial — memory extraction mechanism stores user facts and context summaries across sessions, which influences future agent behavior. This is not learning in the ML sense, but behavioral adaptation via persistent context.

- Memory is updated after each conversation via LLM-based fact extraction — evidence: `agents/memory/updater.py:269-333`
- Extracted facts are injected into system prompt in future sessions — evidence: `agents/lead_agent/prompt.py:351-379`
- The agent adapts responses based on stored user context (work context, personal context, preferences as facts)

**Mechanism:**

| **Mechanism** | **Implementation** | **Evidence** |
|---|---|---|
| Fact extraction from conversations | `MemoryUpdater.update_memory()` calls LLM → JSON → `_apply_updates()` | `agents/memory/updater.py:269-412` |
| User context summarization | `workContext`, `personalContext`, `topOfMind` summaries updated when LLM flags `shouldUpdate: true` | `updater.py:354-362` |
| History summarization | `recentMonths`, `earlierContext`, `longTermBackground` updated similarly | `updater.py:364-371` |
| Context injection | Memory injected as `<memory>` block in system prompt each session | `agents/lead_agent/prompt.py:351-379` |

**Explicit reinforcement (thumbs up/down, reward signals):** No implementation — no feedback capture, no rating UI, no reward signal

**Versioning of learned strategies:** No implementation — memory.json has `"version": "1.0"` schema version but no behavioral strategy versioning; no rollback mechanism beyond restoring `memory.json` from backup

**Scope of adaptation:**
- Global by default — `memory.json` shared across all threads and sessions
- Per-agent scope available: `agent_name` parameter routes to separate memory file — evidence: `agents/memory/storage.py:76-86`
- Not scoped per-user or per-task-type (no user identity model)

**Online vs offline learning:**
- Offline — memory updates run in background thread via debounced queue (30s default) after conversation ends
- No model weight updates; adaptation is prompt-level only

**Safeguards against incorrect patterns:**
- Confidence threshold: facts below `fact_confidence_threshold` (default: 0.7) are discarded — evidence: `updater.py:383-384`
- Max facts limit: sorted by confidence, lowest-confidence facts pruned when `max_facts` (100) exceeded — evidence: `updater.py:404-410`
- Upload mention filtering: sentences about session-scoped uploaded files stripped before saving — evidence: `updater.py:319-323`
- No human review gate — all LLM-extracted facts passing confidence threshold are auto-applied
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
# Cluster F: Ops

## SECTION 11: ERROR HANDLING AND RESILIENCE

**Retry strategy for external services:**
- LLM calls: `max_retries` configured per model in `config.yaml` (e.g., `max_retries: 2`); handled by LangChain provider SDK internally — evidence: `config.example.yaml:62`
- Subagent execution: No retry — `SubagentExecutor._aexecute()` catches all exceptions → `SubagentStatus.FAILED` with error message returned — evidence: `subagents/executor.py:343-348`
- MCP tools: No retry in DeerFlow code; transient errors surface as ToolMessage errors
- Memory updates: No retry — `MemoryUpdater.update_memory()` catches exceptions, logs, returns `False` — evidence: `agents/memory/updater.py:326-332`
- Tool calls: No retry — `ToolErrorHandlingMiddleware` converts exceptions to error ToolMessages, agent decides next action

**Circuit breaker:** No implementation — no circuit breaker pattern anywhere in codebase

**Error taxonomy:**

| **Category** | **Handling** |
|---|---|
| Tool execution exceptions | Caught by `ToolErrorHandlingMiddleware` → error ToolMessage (agent continues) |
| Guardrail denial | `GuardrailMiddleware` returns denied ToolMessage (agent continues) |
| LangGraph `GraphBubbleUp` (interrupt/pause signals) | Re-raised to preserve control flow — evidence: `tool_error_handling_middleware.py:46-47` |
| Subagent timeout (900s default) | `FuturesTimeoutError` → `SubagentStatus.TIMED_OUT` |
| MCP tool failure | Error ToolMessage via ToolErrorHandlingMiddleware |
| Memory update failure | Logged + `False` returned; agent continues unaffected |
| Config validation errors | Raised at startup; no runtime recovery |
| Model not found | `ValueError` raised in `make_lead_agent()` |
| JSON parse error in memory | Caught, warning logged, `False` returned |

**Fallback agent / fallback tool:**
- No fallback agent — model selection falls back to default model when requested model not found
- No fallback tool — failed tools produce error ToolMessages; agent re-plans

**Silent failures:**
- Memory update failures: caught and logged at WARNING/EXCEPTION level, `False` returned silently — evidence: `updater.py:326-332`
- MCP tool initialization failures: logged at ERROR level, tools skipped — evidence: `tools/tools.py:113-116`
- ACP tool load failures: logged at WARNING, skipped — evidence: `tools/tools.py:128-129`
- LangSmith tracer attachment failure: WARNING logged, continues without tracing — evidence: `models/factory.py:90-93`
- `GuardrailMiddleware` provider error with `fail_closed=False`: WARNING logged, tool allowed through

**Idempotency:**
- Memory saves: atomic via temp-file-then-rename — evidence: `agents/memory/storage.py:142-146`
- State writes: LangGraph checkpointer handles idempotency via thread_id + checkpoint versioning
- LLM calls: Not idempotent by design; no deduplication
- Tool calls: Not idempotent; no deduplication or at-most-once semantics

**Recovery strategy after failure:**
- Agent receives error ToolMessage → LLM decides: retry different tool, skip, or produce final answer
- No automatic rollback to checkpoint
- No replanning framework — agent adapts ad-hoc

**Partial progress preservation:**
- LangGraph checkpointer saves state after each turn — thread can be resumed from last checkpoint
- Subagent results: partially completed results available via `_background_tasks` dict
- No dead letter queue; failed tasks logged to application logs only

---

## SECTION 12: EVALUATION AND TESTING

**Test types:**
- Unit tests: ~70 test files covering all subsystems — evidence: `backend/tests/`
- Integration tests (live): `test_client_live.py`, `test_create_deerflow_agent_live.py` — require `config.yaml`, run separately
- End-to-end: `test_client_e2e.py`
- Regression tests: `test_harness_boundary.py` (import firewall), `test_docker_sandbox_mode_detection.py`, `test_provisioner_kubeconfig.py`
- CI: `backend-unit-tests.yml` runs `make test` on all PRs and pushes to main

**Mocking strategy:**

| **Dependency** | **Approach** |
|---|---|
| `deerflow.subagents.executor` | Pre-mocked in `conftest.py` via `sys.modules` injection (circular import prevention) — evidence: `tests/conftest.py:26-33` |
| LLM calls | `MagicMock` / patch per test file; no real LLM calls in unit tests |
| File system | Temp directories via `tmp_path` fixture (pytest) |
| Config / `get_app_config()` | Patched or test config files per test |
| External APIs (Tavily, Jina, etc.) | Mocked with `unittest.mock.patch` |
| LangGraph server | Mocked in unit tests; real server used in live tests |

**Test data:**
- Inline fixtures and `MagicMock` instances; no dedicated fixture factories
- `conftest.py` only configures path and circular import mock — no shared fixtures

**LLM calls in tests:**
- Mocked in unit tests — no real model invocations
- Live tests (`test_client_live.py`) require real `config.yaml` with valid API keys

**Agent flow tests:**
- `test_client.py`: 77 unit tests including `TestGatewayConformance` — validates all client return types against Gateway Pydantic response models
- `test_create_deerflow_agent.py`: agent creation flow
- `test_subagent_executor.py`: subagent execution engine

**Agent response quality:**
- No LLM-as-judge evaluation
- No response quality metrics
- Quality assessed via deterministic assertions on output structure (e.g., Pydantic model conformance)

---

## SECTION 13: DEPLOYMENT AND SCALING

**Deployment strategy:**
- Docker Compose — no rolling/blue-green/canary configuration
- Services: nginx (reverse proxy) + frontend (Next.js) + gateway (FastAPI) + langgraph (LangGraph server) + provisioner (optional, Kubernetes sandbox)
- Evidence: `docker/docker-compose.yaml`

**Service configuration:**
- Gateway: `uvicorn --workers 2` — 2 process workers — evidence: `docker-compose.yaml:71`
- LangGraph: `langgraph dev --n-jobs-per-worker 10` — 10 concurrent jobs per worker — evidence: `docker-compose.yaml:122`
- Nginx: single instance reverse proxy; routes `/api/langgraph/*` → port 2024, `/api/*` → port 8001, `/` → port 3000

**Autoscaling:**
- No autoscaling configuration — fixed workers
- Subagent concurrency: capped at `MAX_CONCURRENT_SUBAGENTS = 3` via `SubagentLimitMiddleware` + 3-worker thread pools

**Containerization:**
- Backend: single `backend/Dockerfile` used for both gateway and langgraph services
- Frontend: `frontend/Dockerfile` with multi-stage build (prod target)
- Base image: built with `ghcr.io/astral-sh/uv:0.7.20` for dependency management
- Docker-in-Docker (DooD): Docker socket mounted for `AioSandboxProvider` sandbox containers
- Evidence: `docker/docker-compose.yaml:66-108`

**Config management:**
- `config.yaml` and `extensions_config.json` mounted as read-only volumes; reloaded via mtime detection without restart
- `DEER_FLOW_HOME`, `DEER_FLOW_CONFIG_PATH`, `DEER_FLOW_EXTENSIONS_CONFIG_PATH`, `DEER_FLOW_DOCKER_SOCKET`, `DEER_FLOW_REPO_ROOT`, `BETTER_AUTH_SECRET` as key environment variables
- Evidence: `docker/docker-compose.yaml:1-20`

**Observability:**
- LangSmith tracing: optional, enabled via `LANGSMITH_TRACING=true` + `LANGSMITH_API_KEY` in `.env`
- Token usage tracking: optional `TokenUsageMiddleware` logs at INFO level
- Structured logging via Python `logging` module; `log_level` configurable in `config.yaml`
- No metrics endpoint, no OpenTelemetry integration
# Synthesis

## SECTION 14: DECISION EXTRACTION AND CLASSIFICATION

### 14A — Decision Extraction

| # | Decision | Classification | Section ref |
|---|---|---|---|
| 1 | **Upload mentions stripped from memory before saving.** Files uploaded in a session are session-scoped; if the agent stores "user uploaded file X" as a long-term fact, it will try to access that file in a future session and hallucinate or fail. A regex filter removes upload-event sentences before any memory persist call. | UNIVERSAL | Section 5 |
| 2 | **Memory updates run in a background thread with a debounce queue (30s), not inline after each message.** Inline memory updates would add LLM latency to every user interaction. The debounce batches rapid back-and-forth turns into a single update call and avoids blocking the user. | ENGINEERING | Section 5 |
| 3 | **Empty dict `{}` sent as `viewed_images` update is a sentinel that clears all image state.** The standard LangGraph reducer merges dicts — it has no native "clear" signal. Using an empty dict as a special-case clear allows middlewares to reset image state without needing a separate flag field or a custom reducer that distinguishes "no update" from "explicit clear." | FRAMEWORK-SPECIFIC | Section 4 |
| 4 | **`LoopDetectionMiddleware` injects a `HumanMessage` (not `SystemMessage`) for mid-conversation warnings.** Anthropic models require system messages only at the conversation start; injecting a `SystemMessage` mid-conversation crashes `langchain_anthropic`'s formatter. Using `HumanMessage` works universally across all providers. | UNIVERSAL | Section 9 |
| 5 | **The harness (`deerflow.*`) is a publishable package that never imports from the app layer (`app.*`).** This boundary is enforced by a CI test (`test_harness_boundary.py`). Without the boundary, the agent framework becomes entangled with the HTTP gateway, making it impossible to use the framework in embedded mode (`DeerFlowClient`) or to publish it as a standalone library. | ENGINEERING | Section 1 (CLAUDE.md) |
| 6 | **MCP tools are loaded lazily at first use with mtime-based cache invalidation, not at startup.** Loading at startup would fail if the MCP server is not running yet. mtime invalidation ensures that config changes made via the Gateway API (in a separate process) are reflected by the LangGraph server without a restart. | ENGINEERING | Section 6 |
| 7 | **When `tool_search` is enabled, MCP tool schemas are withheld from the LLM context and only names are listed in the system prompt.** With many MCP servers, loading all tool schemas inflates the context window significantly and degrades tool selection accuracy. The agent loads a tool's schema on-demand via `tool_search`. | UNIVERSAL | Sections 1, 6 |
| 8 | **Subagents inherit sandbox state and thread_data from the parent agent by passing them through initial state.** Without this, subagents would create their own sandbox context and would not share the parent's workspace directories, breaking file handoffs between lead agent and subagents. | FRAMEWORK-SPECIFIC | Sections 3, 4 |
| 9 | **`SubagentLimitMiddleware` truncates excess `task` tool calls from the model response (application-layer enforcement), not at the API level.** LLM APIs do not enforce per-response tool call count limits. Without application-layer truncation, a model that emits 10 `task` calls would launch 10 concurrent subagent threads — more than the thread pool supports — silently losing work. | UNIVERSAL | Section 3 |
| 10 | **Subagent execution uses a dual thread pool: `_scheduler_pool` (3 workers) submits to `_execution_pool` (3 workers) with a timeout wrapper.** A single pool would deadlock: the scheduler thread waiting for `execution_future.result(timeout=N)` would hold a worker while the execution work has no thread to run on. The separation ensures the scheduler can always wait without blocking executor slots. | ENGINEERING | Section 3 |
| 11 | **Memory facts have a confidence threshold (default 0.7) below which they are discarded, and when the max (100) is exceeded, lowest-confidence facts are pruned.** Without confidence filtering, low-quality LLM extractions pollute long-term memory and get injected into system prompts, degrading future response quality. | UNIVERSAL | Section 5 |
| 12 | **Active web content artifacts (HTML/XHTML/SVG) are always served as `Content-Disposition: attachment`, never inline.** An agent generating HTML could accidentally or maliciously produce scripts that execute in the application's origin if served inline. Forcing download prevents XSS via agent-generated artifacts. | ENGINEERING | Section 9 |
| 13 | **`DanglingToolCallMiddleware` injects placeholder `ToolMessages` for tool calls that were interrupted (lack responses).** When a user interrupts an agent mid-execution, the message history may contain `AIMessage.tool_calls` with no matching `ToolMessage` response. Most LLM APIs reject such malformed histories. The middleware patches the history before the next model call. | UNIVERSAL | Section 1 (CLAUDE.md) |
| 14 | **Memory file saves use temp-file-then-rename for atomic writes.** If the process dies mid-write to `memory.json` directly, the file is left partially written and future reads return corrupt JSON. The temp-file approach ensures reads always see a complete, valid file. | ENGINEERING | Section 4 |
| 15 | **Config values starting with `$` are resolved as environment variables at read time.** This allows sensitive credentials to be stored in `.env` or environment variables rather than in the config file itself, keeping the config file safe to commit while still supporting per-environment overrides. | ENGINEERING | Section 1 |
| 16 | **`AppConfig` caches the parsed config but auto-reloads when the file's mtime increases.** The Gateway API and LangGraph server run as separate processes that both read `config.yaml`. Without mtime-based reload, a config change via the Gateway API would not be seen by the LangGraph server until restart. | ENGINEERING | Section 1 (CLAUDE.md) |
| 17 | **`GuardrailMiddleware` has a `fail_closed` mode (default: `true`) — provider errors block the tool call.** If the guardrail provider crashes and the default was `fail_open`, all tool calls would be allowed through, silently disabling the guardrail. Fail-closed is the safer default for security-sensitive deployments. | ENGINEERING | Section 9 |
| 18 | **The subagent executor runs `asyncio.run()` inside a `ThreadPoolExecutor` worker to support async MCP tools.** MCP tools are inherently async (network I/O). Subagents run in thread pool workers which have no event loop. Without `asyncio.run()`, async tool calls would fail. | ENGINEERING | Section 3 |
| 19 | **Per-agent memory isolation is available via an `agent_name` key that maps to a separate `memory.json` file.** Without isolation, a custom agent's memory would pollute the global memory shared with the lead agent. The file-per-agent pattern prevents cross-contamination without requiring a database. | ENGINEERING | Section 5 |
| 20 | **Gemini thinking mode requires `PatchedChatOpenAI` (not the standard `ChatOpenAI`) to preserve `thought_signature` values across multi-turn tool-call conversations.** The Gemini API requires these signatures to be echoed back; `langchain_openai:ChatOpenAI` strips them. Without the patch, Gemini thinking mode breaks on the second tool call. | FRAMEWORK-SPECIFIC | Section 1 |

---

### 14B — Anti-patterns

| # | What's wrong | What goes wrong in production | What the correct approach looks like | Section ref |
|---|---|---|---|---|
| 1 | **No rate limiting at API, tool, or LLM call level.** | A runaway agent or malicious user can saturate the LLM provider quota, exhaust thread pool workers, or spin up unlimited subagents via repeated API calls. Since only 3 subagent slots exist but unlimited `task` calls can arrive from the API, a denial of service against the agent is trivially constructable. | Implement request-level rate limiting at the nginx or FastAPI layer, and add per-user or per-thread concurrency caps beyond `SubagentLimitMiddleware`. | Section 10 |
| 2 | **No prompt injection detection or sanitization.** | Malicious content in uploaded files, web search results, or MCP tool outputs can instruct the agent to exfiltrate data, call unauthorized tools, or bypass clarification gates. Memory injection makes this worse — a successful injection in one session may persist into future sessions via memory facts. | Add an input normalization step that escapes or strips XML-like tags from tool results before they enter the message history, and consider a lightweight prompt-injection classifier on user inputs. | Sections 9, 5 |
| 3 | **Global `memory.json` shared across all threads with no per-user scoping.** | In a multi-user deployment (e.g., public-facing IM channel bots), one user's private facts (work context, personal preferences) are stored in the same file as another user's. All users see each other's injected memory context, causing privacy leakage and degraded personalization. | Add user identity to the memory storage key — either a per-user file path or a user-keyed section within the store. | Section 5 |

---

### 14C — Design Trade-offs

| Decision point | Choose option A when... | Choose option B when... | This codebase chose | Observed consequence |
|---|---|---|---|---|
| **Memory scope: global vs per-thread** | Single user, all sessions should share context; memory improves over time | Multi-user; users need private memory; per-session memory prevents cross-contamination | Global (single `memory.json`) | Simple to implement; becomes a privacy liability in multi-user deployments |
| **Sandbox: local (host) vs isolated container** | Trusted single-user local workflow; fastest execution; no container overhead | Untrusted code, multi-user, or production environment requiring isolation | Local by default (`allow_host_bash: false` restricts bash), with AIO (Docker) option | Local is safe for file ops but disables bash; Docker option adds setup complexity |
| **Tool schemas: eager binding vs deferred via tool_search** | Small tool set (≤10–15 tools); context window not a concern | Large MCP tool set; context window costs outweigh tool discovery latency | Eager (default); deferred opt-in via `tool_search.enabled` | Eager loading works well for small configs; large MCP deployments need to enable `tool_search` explicitly |
| **Subagent model: inherit vs dedicated** | Parent model is capable and cost of subagent calls is not a concern | Subagents do simpler work; cheaper/faster model appropriate | Inherit parent model by default; per-subagent override available | Inheriting ensures capability parity; adds token cost when parent is a high-end model |
| **Memory injection: full history vs structured facts** | User needs agent to recall exact prior conversation details | Long-term patterns and preferences matter more than episodic recall | Structured facts + context summaries (not raw history) | Reduces token cost; loses episodic detail; agent cannot recall specific prior conversation exchanges |
