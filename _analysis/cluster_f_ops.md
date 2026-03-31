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
