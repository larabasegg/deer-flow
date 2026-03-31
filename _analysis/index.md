# Codebase Index

## Project Metadata

- **Project name**: DeerFlow
- **Stated purpose**: "LangGraph-based AI agent system with sandbox execution capabilities"
- **Use case domain**: AI super-agent platform — general-purpose agentic assistant with web research, code execution, memory, and multi-agent delegation
- **Project type**: Open-source product (full-stack, deployable)
- **Author(s) / organization**: Community-driven open source project

## Framework

- **Name**: LangGraph (via `langchain.agents` + `langgraph-sdk`)
- **Version**: `langgraph-sdk>=0.1.51` (backend dependency)
- **Provider SDKs**:
  - `langchain_openai` (ChatOpenAI, patched variants)
  - `langchain_anthropic` (ChatAnthropic)
  - `langchain_google_genai` (ChatGoogleGenerativeAI)
  - `langchain-mcp-adapters` (MultiServerMCPClient)
  - `langchain.agents` (create_agent, AgentMiddleware, SummarizationMiddleware, AgentState)
- **Runtime infra**: FastAPI (gateway), Uvicorn, Nginx (reverse proxy), SQLite/PostgreSQL (checkpointer), Docker (sandbox)

## Directory Map

| Directory | Purpose |
|---|---|
| `backend/` | Python backend — agent runtime, gateway API, tests |
| `backend/packages/harness/deerflow/` | `deerflow-harness` publishable package — all agent logic |
| `backend/app/` | Application layer — FastAPI gateway, IM channel integrations |
| `backend/tests/` | Unit and integration test suite |
| `frontend/` | Next.js web interface |
| `docker/` | Docker Compose configs, Nginx configs, Provisioner service |
| `skills/` | Agent skills directory (`public/`, `custom/`) |
| `_analysis/` | Analysis outputs (this file and cluster reports) |
| `.claude/` | Claude Code skills registry |

## Key File Registry

| Category | File Path | Purpose |
|---|---|---|
| **Entry point (agent)** | `backend/packages/harness/deerflow/agents/lead_agent/agent.py` | `make_lead_agent()` — LangGraph agent factory registered in `langgraph.json` |
| **LangGraph config** | `backend/langgraph.json` | Registers `lead_agent` graph + custom checkpointer |
| **Thread state** | `backend/packages/harness/deerflow/agents/thread_state.py` | `ThreadState` — extends `AgentState` with sandbox, thread_data, artifacts, todos, etc. |
| **System prompt** | `backend/packages/harness/deerflow/agents/lead_agent/prompt.py` | `apply_prompt_template()` — injects skills, memory, subagent instructions |
| **Middleware chain** | `backend/packages/harness/deerflow/agents/lead_agent/agent.py:208` | `_build_middlewares()` — assembles ordered middleware list |
| **Middleware directory** | `backend/packages/harness/deerflow/agents/middlewares/` | 14 middleware modules (clarification, dangling tool call, loop detection, memory, sandbox audit, subagent limit, thread data, title, todo, token usage, tool error handling, uploads, view image, deferred tool filter) |
| **Tool assembly** | `backend/packages/harness/deerflow/tools/` | `get_available_tools()` — assembles sandbox, builtin, MCP, community, subagent tools |
| **Model factory** | `backend/packages/harness/deerflow/models/factory.py` | `create_chat_model()` — reflection-based LLM instantiation |
| **Config system** | `backend/packages/harness/deerflow/config/app_config.py` | `get_app_config()` — cached, mtime-reloading config |
| **Config example** | `config.example.yaml` | Schema reference + `config_version: 4` |
| **Extensions config** | `extensions_config.example.json` | MCP servers + skills enabled state |
| **Sandbox interface** | `backend/packages/harness/deerflow/sandbox/sandbox.py` | Abstract `Sandbox` — `execute_command`, `read_file`, `write_file`, `list_dir` |
| **Sandbox tools** | `backend/packages/harness/deerflow/sandbox/tools.py` | `bash_tool`, `ls_tool`, `read_file_tool`, `write_file_tool`, `str_replace_tool` |
| **Local sandbox** | `backend/packages/harness/deerflow/sandbox/local/local_sandbox_provider.py` | Singleton local FS provider |
| **AIO sandbox** | `backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py` | Docker-based isolation provider |
| **Subagent executor** | `backend/packages/harness/deerflow/subagents/executor.py` | Background dual-thread-pool subagent execution |
| **MCP client** | `backend/packages/harness/deerflow/mcp/client.py` | `MultiServerMCPClient` wrapper, lazy + mtime-cached |
| **Memory updater** | `backend/packages/harness/deerflow/agents/memory/updater.py` | LLM-based fact extraction + atomic file write |
| **Memory queue** | `backend/packages/harness/deerflow/agents/memory/queue.py` | Debounced per-thread memory update queue |
| **Embedded client** | `backend/packages/harness/deerflow/client.py` | `DeerFlowClient` — in-process access, no HTTP |
| **Reflection** | `backend/packages/harness/deerflow/reflection/resolvers.py` | `resolve_variable()`, `resolve_class()` — dynamic module loading |
| **Gateway app** | `backend/app/gateway/app.py` | FastAPI app on port 8001 |
| **Gateway routers** | `backend/app/gateway/routers/` | 11 routers: models, mcp, skills, memory, uploads, threads, artifacts, suggestions, agents, channels, thread_runs, runs |
| **Channels** | `backend/app/channels/` | IM platform integrations (Feishu, Slack, Telegram) + message bus + store |
| **Skills loader** | `backend/packages/harness/deerflow/skills/loader.py` | Recursive scan of `skills/{public,custom}/SKILL.md` |
| **Skills installer** | `backend/packages/harness/deerflow/skills/installer.py` | ZIP-based `.skill` archive extraction |
| **Guardrails** | `backend/packages/harness/deerflow/guardrails/` | Pluggable `GuardrailProvider` — allowlist builtin + OAP support |
| **Checkpointer** | `backend/packages/harness/deerflow/agents/checkpointer/async_provider.py` | `make_checkpointer()` — memory/sqlite/postgres |
| **Runtime store** | `backend/packages/harness/deerflow/runtime/store/` | SQLite-backed KV store |
| **Stream bridge** | `backend/packages/harness/deerflow/runtime/stream_bridge/` | SSE streaming abstraction |
| **Provisioner** | `docker/provisioner/app.py` | Kubernetes sandbox pod provisioner |
| **Env example** | `.env.example` | API key placeholders (Tavily, Jina, LangSmith, etc.) |
| **Docker Compose (prod)** | `docker/docker-compose.yaml` | nginx + frontend + gateway + langgraph + provisioner |
| **CI workflows** | `.github/workflows/backend-unit-tests.yml` | Pytest on PRs |
| **Tests** | `backend/tests/` | 70+ test files covering all subsystems |
