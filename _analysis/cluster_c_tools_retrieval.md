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
