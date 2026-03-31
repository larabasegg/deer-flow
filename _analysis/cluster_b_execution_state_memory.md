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
