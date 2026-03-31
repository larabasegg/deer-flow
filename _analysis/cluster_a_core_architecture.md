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
