# Design Decisions
*Snapshot analysis of current code state as of 2026-03-31.*
*Distinct from `architecture.md` (historical evolution) and `directives.md` (what to avoid from git history).*
*Captures the reasoning behind current architectural choices, trade-offs made, and non-obvious system behaviors.*

## Architectural patterns

### Multi-process filesystem coordination via mtime invalidation
**Concern:** LangGraph Server and Gateway run as separate OS processes sharing config and state files. Changes written by one process must be visible to the other without restarts.
**Subsystems involved:** Configuration System, Skills & Extensions, Memory Persistence
**How they compose:** `get_app_config()` stats `config.yaml` on every call. `load_skills()` reads `extensions_config.json` directly (bypassing the singleton cache). `FileMemoryStorage.load()` compares mtime before returning cached data. Together they form a pull-based coordination model where file-mtime is the synchronization primitive — no message passing, no shared memory, no explicit invalidation signals.
**What breaks without any one piece:** Without mtime-check in AppConfig, config edits require process restart. Without cache bypass in `load_skills()`, the LangGraph Server would serve stale skill lists after Gateway-side updates. Without mtime-check in memory storage, the agent would inject stale facts after an async memory write completes.
**Scope:** `[codebase]`

### Defence-in-depth for user-supplied content
**Concern:** User-supplied content enters the system at four distinct surfaces: file serving, archive installation, IM delivery, and code execution. Each must independently reject attacks.
**Subsystems involved:** Gateway API Layer, Skills & Extensions, IM Channels Integration, Sandbox Provider Abstraction
**How they compose:** Artifacts endpoint forces download for HTML/SVG (XSS prevention). Skill installer uses double path-traversal check and ZIP bomb protection. IM channel attachment delivery enforces `/mnt/user-data/outputs/` prefix allowlist plus `relative_to()` traversal guard. Sandbox security gates host bash execution behind `allow_host_bash`. No single bypass breaks all four layers simultaneously.
**What breaks without any one piece:** Removing artifact MIME check allows XSS via agent-generated HTML. Removing ZIP traversal check allows skill install to escape the custom/ directory. Removing IM attachment allowlist allows workspace/upload file exfiltration through IM replies. Removing host bash gate restores the pre-D2 security escape.
**Scope:** `[technology]`

### Atomic file I/O as a system-wide consistency primitive
**Concern:** Multiple subsystems persist state to JSON files that must remain consistent under crashes or concurrent writes.
**Subsystems involved:** Memory Persistence, IM Channels Integration, Embedded Client
**How they compose:** All three use the same pattern: write to a `.tmp` file in the same directory as the target, then call `Path.replace()` for an OS-level atomic rename. A crash mid-write leaves the target file intact. The pattern is identical in all three sites.
**What breaks without any one piece:** Memory file corruption would cause the agent to lose all learned facts and start with empty memory. Channel store corruption would cause all IM conversations to lose their thread mappings. Embedded client config mutation corruption would leave extensions in an inconsistent state.
**Scope:** `[technology]`

---

## Embedded Client (DeerFlowClient)

### Decisions

- **`stream()` uses `stream_mode="values"` and reconstructs `messages-tuple` events from state snapshots** `[codebase]` — the HTTP gateway uses `stream_mode=["messages-tuple", "values"]` to receive native per-token events; the embedded client uses `stream_mode="values"` only and synthesizes `messages-tuple` events from cumulative state diffs. Consequence: the client cannot provide true per-token granularity. Each `messages-tuple` event represents a completed message, not a streaming chunk. Evidence: `deerflow/client.py:L362` — `for chunk in self._agent.stream(state, config=config, context=context, stream_mode="values")`

- **Agent cache key excludes `agent_name` — changing agent identity requires an explicit `reset_agent()` call** `[codebase]` — `_ensure_agent()` rebuilds the agent when `(model_name, thinking_enabled, is_plan_mode, subagent_enabled)` changes. `agent_name` is fixed at `__init__` and not part of the key. A developer who creates one client instance and passes different `agent_name` values via overrides will get the original agent. Evidence: `deerflow/client.py:L205-L211` — `key = (cfg.get("model_name"), cfg.get("thinking_enabled"), cfg.get("is_plan_mode"), cfg.get("subagent_enabled"))` (no `agent_name`)

- **`chat()` returns only the LAST AI text event — intermediate AI text segments are discarded** `[codebase]` — if the agent emits multiple text segments in a turn (e.g., a short preamble followed by a long response), `chat()` returns only the last non-empty content. Use `stream()` to capture all. The docstring calls this out explicitly. Evidence: `deerflow/client.py:L442-L448` — `if event.type == "messages-tuple" and event.data.get("type") == "ai": content = event.data.get("content", ""); if content: last_text = content` (overwrites each time)

- **Without a checkpointer, `stream()` is stateless even with a `thread_id`** `[technology]` — `thread_id` is used for file isolation (uploads/outputs directories) only. Without a checkpointer, message history is not persisted between calls. No error is raised; the caller receives a valid first-turn response but subsequent calls have no memory of prior turns. Evidence: `deerflow/client.py:L126-L129` — `Note: Multi-turn conversations require a checkpointer. Without one, each stream()/chat() call is stateless — thread_id is only used for file isolation`

### Non-obvious behaviors

- **`stream()` deduplicates messages using `seen_ids` because `stream_mode="values"` yields cumulative state** `[technology]` — each yielded state snapshot contains all messages from the start of the turn, so without deduplication the same message would be yielded once per downstream state update. Evidence: `deerflow/client.py:L359-L370` — `seen_ids: set[str] = set()` … `if msg_id and msg_id in seen_ids: continue`

- **`_extract_text()` uses a heuristic to detect JSON token chunks and joins them without separators** `[codebase]` — short strings (≤20 chars) containing JSON syntax characters (`{}[]":,`) are treated as token chunks and joined without separators to avoid corrupting JSON payloads. Longer strings are joined with newlines for readability. Evidence: `deerflow/client.py:L288` — `chunk_like = len(content) > 1 and all(isinstance(block, str) and len(block) <= 20 and any(ch in block for ch in '{}[]":,') for block in content)`

- **`update_mcp_config()` and `update_skill()` automatically call `reset_agent()` after writing config** `[codebase]` — any method that changes extensions config invalidates the agent so the next call rebuilds with the updated tool set and system prompt. Callers don't need to call `reset_agent()` manually after these mutations. (Confirmed in CLAUDE.md docstring note: `update_mcp_config()` and `update_skill()` automatically invalidate the cached agent.)

---

## Skills & Extensions System

### Decisions

- **`load_skills()` reads `ExtensionsConfig.from_file()` directly, bypassing the singleton cache** `[codebase]` — comment explicitly states this is to ensure the LangGraph Server (separate process) always sees Gateway-written config changes. Using `get_extensions_config()` would return a stale cached view. Consequence: every `load_skills()` call incurs a disk read of `extensions_config.json`. Evidence: `deerflow/skills/loader.py:L80-L88` — `# NOTE: We use ExtensionsConfig.from_file() instead of get_extensions_config() to always read the latest configuration from disk.`

- **ZIP extraction uses a two-layer path traversal defense: upfront name check AND post-normalization `is_relative_to` check** `[technology]` — `is_unsafe_zip_member()` rejects `..` and absolute paths on both POSIX and Windows, but after normalization the resolved path is also checked against `dest_root`. This guards against edge cases where path normalization could still produce an escaping path after the initial filter. Evidence: `deerflow/skills/installer.py:L99-L102` — `normalized_name = posixpath.normpath(...); member_path = dest_root.joinpath(...); if not member_path.resolve().is_relative_to(dest_root): raise ValueError(...)`

- **`get_skills_root_path()` calculates its path by navigating 5 parent directories from `loader.py`'s location** `[codebase]` — rather than reading config or an environment variable, the function derives the skills path from its own file location. This breaks if `loader.py` is installed into a system package location (e.g., `site-packages`). Evidence: `deerflow/skills/loader.py:L18-L22` — `backend_dir = Path(__file__).resolve().parent.parent.parent.parent.parent; skills_dir = backend_dir.parent / "skills"`

### Trade-offs

| Decision point | Choose A when... | Choose B when... | This codebase chose | Structural consequence |
|---|---|---|---|---|
| Extensions config read in `load_skills()` | Intra-process, cache-coherent | Cross-process, must read latest | Always read from disk (bypass cache) | Every skill load incurs a disk read; cache coherence across processes is maintained at the cost of I/O |
| Skill archive format | Single-file skills | Multi-file skills with assets | ZIP with `.skill` extension | Skills must be packaged as archives; single Markdown files cannot be installed via the API |

### Non-obvious behaviors

- **Symlinks inside `.skill` ZIP archives are silently skipped, not extracted** `[technology]` — `is_symlink_member()` detects symlinks via Unix mode bits in the ZipInfo `external_attr`. Symlinks are skipped with a warning, which may leave the installed skill missing files if it relies on symlinks. Evidence: `deerflow/skills/installer.py:L95-L97` — `if is_symlink_member(info): logger.warning("Skipping symlink entry in skill archive: %s", info.filename); continue`

- **macOS-created archives containing `__MACOSX` metadata directories are handled transparently** `[codebase]` — `resolve_skill_dir_from_archive()` and `should_ignore_archive_entry()` filter out `__MACOSX` and dotfiles before looking for the skill root directory. Archives created with macOS Finder install correctly without errors. Evidence: `deerflow/skills/installer.py:L49-L51` — `def should_ignore_archive_entry(path: Path) -> bool: return path.name.startswith(".") or path.name == "__MACOSX"`

- **ZIP bomb protection enforces a 512MB total uncompressed size limit during streaming extraction** `[technology]` — unlike standard `zipfile.extractall()`, `safe_extract_skill_archive()` streams and accumulates the written bytes, raising before completion if the limit is exceeded. The ZIP file itself may be small; only the uncompressed content matters. Evidence: `deerflow/skills/installer.py:L76,L111-L113` — `max_total_size: int = 512 * 1024 * 1024` … `if total_written > max_total_size: raise ValueError("Skill archive is too large or appears highly compressed.")`

---

## Sandbox Provider Abstraction

### Decisions

- **`LocalSandboxProvider` uses a single module-level `_singleton` — all threads share the same execution environment** `[codebase]` — `acquire()` creates one `LocalSandbox` instance for the process lifetime, regardless of `thread_id`. Per-thread filesystem isolation in local mode is handled by the virtual path system (each thread gets its own `/mnt/user-data/...` directory), not by separate sandbox instances. A developer expecting per-thread sandbox isolation at the execution level will not find it. Evidence: `deerflow/sandbox/local/local_sandbox_provider.py:L9,L45-L49` — `_singleton: LocalSandbox | None = None` … `if _singleton is None: _singleton = LocalSandbox("local", ...)` (module global, not per-thread)

- **`LocalSandboxProvider.release()` is intentionally a no-op** `[codebase]` — `SandboxMiddleware` calls `release()` after each agent turn, but the comment explicitly states this is intentional. The sandbox persists for the process lifetime to allow cross-turn reuse. Docker-based providers clean up during `shutdown()`. Developers following the `acquire/release` contract and expecting cleanup at release time will be surprised. Evidence: `deerflow/sandbox/local/local_sandbox_provider.py:L58-L64` — `def release(self, sandbox_id: str) -> None: # LocalSandbox uses singleton pattern - no cleanup needed.`

- **`reset_sandbox_provider()` clears the provider singleton but NOT the underlying `LocalSandbox` module-level `_singleton`** `[codebase]` — after `reset_sandbox_provider()`, the next `get_sandbox_provider()` creates a new `LocalSandboxProvider` instance with fresh path mappings, but `acquire()` will still return the existing `_singleton` LocalSandbox. Tests that call `reset_sandbox_provider()` between runs may not get a clean sandbox if the module-level singleton is not also reset. Evidence: `deerflow/sandbox/local/local_sandbox_provider.py:L9` (`_singleton` is module-level, not `LocalSandboxProvider` instance-level) vs `deerflow/sandbox/sandbox_provider.py:L59-L71` (resets only `_default_sandbox_provider`)

- **Command path resolution uses regex substitution with a path-boundary lookahead to prevent over-matching** `[codebase]` — `/mnt/skills` in a command is replaced with the host path, but the regex requires the match to end at `/`, end of string, or a shell separator character. Without this, `/mnt/skills-extra` would accidentally match `/mnt/skills` and corrupt the command. Evidence: `deerflow/sandbox/local/local_sandbox.py:L160` — `r"(?=/|$|[\s\"';&|<>()])"` lookahead in path pattern

- **Host OSErrors are re-raised with the virtual path, not the resolved host path** `[codebase]` — `read_file`, `write_file`, and `update_file` catch `OSError` from the resolved host path and re-raise with the container path argument. This prevents host filesystem paths (e.g., `/Users/lara/.deer-flow/...`) from appearing in agent-visible error messages. Evidence: `deerflow/sandbox/local/local_sandbox.py:L247` — `raise type(e)(e.errno, e.strerror, path) from None` (uses original `path`, not `resolved_path`)

### Trade-offs

| Decision point | Choose A when... | Choose B when... | This codebase chose | Structural consequence |
|---|---|---|---|---|
| Per-thread execution isolation | Security boundary needed | Performance and simplicity matter | Shared singleton for LocalSandboxProvider, per-thread dirs for virtual path | All threads share the same host process; a runaway command can affect other threads' files |
| Path resolution in commands | Paths always well-formed | Paths may have unusual characters | Regex substitution | Paths with spaces or regex-special characters in container paths could cause substitution failures |

### Non-obvious behaviors

- **`execute_command` reverse-resolves host paths back to container paths in the output** `[codebase]` — after running a command, any host filesystem paths appearing in stdout/stderr (e.g., from `ls` or `find`) are substituted back to their container path equivalents. The agent always sees `/mnt/skills/...` not `/Users/.../skills/...` in command output. Evidence: `deerflow/sandbox/local/local_sandbox.py:L232` — `return self._reverse_resolve_paths_in_output(final_output)`

- **`execute_command` merges stdout and stderr into a single return string** `[codebase]` — if both streams are non-empty, stderr is appended to stdout with `\nStd Error:\n` prefix. Callers receive a single string and cannot distinguish which stream a given line came from. Evidence: `deerflow/sandbox/local/local_sandbox.py:L224-L230` — `output += f"\nStd Error:\n{result.stderr}" if output else result.stderr`

- **`is_host_bash_allowed()` always returns `True` for non-`LocalSandboxProvider` configurations** `[technology]` — the `allow_host_bash` guard only applies when the active provider is `LocalSandboxProvider`. Docker-based providers (`AioSandboxProvider`) execute inside containers and don't need the guard. Evidence: `deerflow/sandbox/security.py:L43-L44` — `if not uses_local_sandbox_provider(config): return True`

---

## Memory Persistence Layer

### Decisions

- **The debounce queue replaces earlier updates for the same thread instead of accumulating them** `[technology]` — when `add()` is called for a thread that already has a pending update, the older context is discarded and replaced with the newer one. Consequence: if a thread produces multiple conversation turns within the debounce window, only the most recent turn's messages are used for memory extraction. Evidence: `deerflow/agents/memory/queue.py:L60-L62` — `self._queue = [c for c in self._queue if c.thread_id != thread_id]; self._queue.append(context)`

- **Every call to `add()` resets the debounce timer — the window restarts from zero on each enqueue** `[technology]` — a rapidly chatting user will continuously delay memory processing. The debounce fires only after `debounce_seconds` of inactivity. This prevents spamming the LLM with partial conversations but can delay persistence indefinitely under high load. Evidence: `deerflow/agents/memory/queue.py:L64-L65` — `self._reset_timer()` (called inside the `with self._lock` block after every `add`)

- **Memory LLM calls run in a `threading.Timer` daemon thread, not an asyncio task** `[technology]` — the memory update is a long-running synchronous LLM call. Running it in a daemon thread keeps the asyncio event loop unblocked. Daemon threads are killed without cleanup on ungraceful process exit, so in-flight memory updates at shutdown are silently lost. Evidence: `deerflow/agents/memory/queue.py:L78-L83` — `self._timer = threading.Timer(config.debounce_seconds, self._process_queue); self._timer.daemon = True`

- **Upload-related sentences are stripped from memory AFTER LLM extraction, not filtered from input** `[codebase]` — `_strip_upload_mentions_from_memory()` runs on the updated memory dict before saving. If the scrubbing regex is too narrow, upload paths persist in memory and the agent will try to access those files in future sessions where they don't exist. Evidence: `deerflow/agents/memory/updater.py:L322-L325` — `updated_memory = _strip_upload_mentions_from_memory(updated_memory)` (after `_apply_updates`, before `save`)

- **Fact deduplication uses exact-match on stripped content, not semantic similarity** `[technology]` — `_fact_content_key` normalizes with `.strip()` only. Two semantically identical facts phrased differently (e.g., "User prefers Python" vs "The user likes Python") will both be stored and consume slots toward `max_facts`. Evidence: `deerflow/agents/memory/updater.py:L380-L388` — `existing_fact_keys = {fact_key for fact_key in (_fact_content_key(fact.get("content")) ... }; if fact_key in existing_fact_keys: continue`

### Trade-offs

| Decision point | Choose A when... | Choose B when... | This codebase chose | Structural consequence |
|---|---|---|---|---|
| Debounce queue behavior | All turns must be extracted | Latest conversation state is sufficient | Replace-per-thread | Multi-turn conversations within the debounce window lose earlier turn facts |
| Memory update execution | Blocking sync acceptable | Agent loop must stay responsive | Daemon thread (not async) | Memory updates survive event loop restarts but are lost on ungraceful shutdown |
| Fact deduplication strategy | Semantic meaning matters | Exact-match is sufficient and cheap | Exact stripped-content match | Near-duplicate facts phrased differently accumulate and consume fact limit slots |

### Non-obvious behaviors

- **`memory_file` write uses temp-file atomic rename — a partial write never corrupts the active file** `[technology]` — `save()` writes to `memory.tmp` then calls `temp_path.replace(file_path)`. A crash mid-write leaves the old `memory.json` intact. Evidence: `deerflow/agents/memory/storage.py:L142-L146` — `temp_path = file_path.with_suffix(".tmp"); ... temp_path.replace(file_path)`

- **Memory `load()` is mtime-cached — two agents reading the same file see a consistent snapshot even across writes** `[technology]` — `FileMemoryStorage` stores `(data, mtime)` in `_memory_cache`. A fresh write updates the cache with the new mtime. Callers that hold a reference to the returned dict are holding a snapshot, not a live view. Evidence: `deerflow/agents/memory/storage.py:L103-L119` — `if cached is None or cached[1] != current_mtime: memory_data = self._load_memory_from_file(...); self._memory_cache[agent_name] = (memory_data, current_mtime)`

- **Memory updater always disables thinking mode, even when the global config enables it** `[codebase]` — `MemoryUpdater._get_model()` calls `create_chat_model(name=..., thinking_enabled=False)` unconditionally. Memory extraction prompts are structured JSON extraction tasks that don't benefit from thinking and would be more expensive. Evidence: `deerflow/agents/memory/updater.py:L267` — `return create_chat_model(name=model_name, thinking_enabled=False)`

- **`_process_queue` re-schedules itself if already processing instead of dropping the trigger** `[technology]` — if the LLM call takes longer than `debounce_seconds`, a second timer fires while the first is still running. Rather than running concurrently, the second trigger resets the timer for a future attempt. Evidence: `deerflow/agents/memory/queue.py:L93-L96` — `if self._processing: self._reset_timer(); return`

### Structural anti-patterns

- **`max_facts` enforcement sorts by confidence and truncates — lowest-confidence facts are silently deleted without logging** — when `len(facts) > max_facts`, the oldest/least-confident facts are discarded with no notification. Facts may represent important persistent context (goals, preferences) that happens to have been assigned low initial confidence by the LLM. Correct structure: log the discarded fact IDs at INFO level so operators can detect fact loss. Evidence: `deerflow/agents/memory/updater.py:L404-L411` — `current_memory["facts"] = sorted(... reverse=True)[: config.max_facts]` (no logging of removed facts)

---

## Configuration System

### Decisions

- **Sub-system configs (memory, title, summarization, subagents, etc.) are stored as module-level globals loaded as side effects of `AppConfig.from_file()`, not as fields on `AppConfig`** `[codebase]` — code in `from_file()` calls `load_memory_config_from_dict()`, `load_title_config_from_dict()`, etc. as side effects; these populate module-level singletons in their respective modules. Access is via `get_memory_config()`, not `get_app_config().memory`. A developer who constructs `AppConfig` directly (bypassing `from_file()`) or reloads only specific sub-configs will have stale or inconsistent sub-system state. Evidence: `deerflow/config/app_config.py:L96-L127` — `if "memory" in config_data: load_memory_config_from_dict(config_data["memory"])` … (repeated pattern for each sub-system)

- **Extensions env variable resolution silently replaces unresolved `$VAR` with empty string; main config raises `ValueError`** `[codebase]` — `AppConfig.resolve_env_variables()` raises on missing env vars; `ExtensionsConfig.resolve_env_variables()` replaces with `""`. An MCP server with `env: {API_KEY: $MISSING_KEY}` starts with an empty API key and fails at runtime with a connection error — no startup warning. Evidence: `deerflow/config/extensions_config.py:L162-L164` — `config[key] = ""  # Unresolved placeholder — store empty string so downstream consumers ... don't receive the literal "$VAR" token`

- **`set_app_config()` bypasses mtime-based auto-reload permanently via `_app_config_is_custom = True`** `[technology]` — once a custom config is injected, `get_app_config()` returns it unconditionally — the mtime check is skipped. This allows tests to inject configs without them being overwritten by disk reads. But it also means a process that calls `set_app_config()` at startup will never pick up `config.yaml` edits until `reset_app_config()` is called. Evidence: `deerflow/config/app_config.py:L279-L281` — `if _app_config is not None and _app_config_is_custom: return _app_config`

- **`AppConfig` uses `extra="allow"` — unknown keys in `config.yaml` are silently accepted** `[technology]` — Pydantic does not raise on unrecognized config keys, so a typo (e.g., `memroy:` instead of `memory:`) produces no warning. The misspelled section is ignored. Evidence: `deerflow/config/app_config.py:L43` — `model_config = ConfigDict(extra="allow", frozen=False)`

- **New skills default to enabled for `public` and `custom` categories when absent from `extensions_config.json`** `[codebase]` — `is_skill_enabled()` returns `True` for any skill in the `public` or `custom` category with no entry in the config file. Skills must be explicitly disabled; an unknown skill is enabled by default. Evidence: `deerflow/config/extensions_config.py:L197-L199` — `if skill_config is None: return skill_category in ("public", "custom")`

### Trade-offs

| Decision point | Choose A when... | Choose B when... | This codebase chose | Structural consequence |
|---|---|---|---|---|
| Sub-system config storage | Unified access via one object | Modules access their config without importing main config | Module-level singletons loaded as side effects of `from_file()` | Two distinct access patterns: `get_app_config().models` vs `get_memory_config()`; constructing `AppConfig` directly skips sub-system initialization |
| Unresolved env vars in extensions | Fail fast at startup | Don't break startup for optional integrations | Silent empty-string replacement | Misconfigured MCP server keys cause runtime failures, not startup errors |
| Config auto-reload trigger | Scheduled polling | On-demand per-call mtime check | mtime comparison on every `get_app_config()` call | Every config access incurs an `os.stat()` syscall; reload is file-mtime-based so in-memory edits are invisible |

### Non-obvious behaviors

- **`ExtensionsConfig` uses `mcpServers` (camelCase) as the JSON file key but exposes it as `mcp_servers` (snake_case) via a Pydantic alias** `[codebase]` — code that writes to `extensions_config.json` directly must use `mcpServers`. Code reading the Python object uses `mcp_servers`. The serialization alias `populate_by_name=True` means both names work in Python but only `mcpServers` is valid JSON. Evidence: `deerflow/config/extensions_config.py:L58-L63` — `mcp_servers: dict[str, McpServerConfig] = Field(... alias="mcpServers")`

- **`_check_config_version` searches up to 5 parent directories for `config.example.yaml`** `[codebase]` — version comparison requires finding the example file. If the working directory is deeply nested (e.g., inside a Makefile task), the example file is still found. If not found within 5 levels, version checking silently skips. Evidence: `deerflow/config/app_config.py:L154-L164` — `for _ in range(5): candidate = search_dir / "config.example.yaml"; if candidate.exists(): example_path = candidate; break`

- **`load_dotenv()` is called at module import time — environment is loaded before config resolution** `[technology]` — importing `deerflow.config.app_config` triggers `.env` file loading as a side effect. In tests or tools that import this module, `.env` from the working directory is automatically applied. Evidence: `deerflow/config/app_config.py:L26` — `load_dotenv()`

---

## IM Channels Integration

### Decisions

- **Custom agents in IM channels are routed as `lead_agent + agent_name context`, not separate LangGraph assistants** `[codebase]` — when `assistant_id` is set in channel config (e.g., `assistant_id: "researcher"`), the value is normalized (lowercase, underscores→hyphens) and injected into the run context as `agent_name`, while the LangGraph `assistant_id` passed to `runs.wait/stream` remains `"lead_agent"`. All IM runs appear identical in LangGraph Server monitoring; the custom agent is invisible at the run level. Evidence: `app/channels/manager.py:L412-L414` — `run_context.setdefault("agent_name", _normalize_custom_agent_name(assistant_id)); assistant_id = DEFAULT_ASSISTANT_ID`

- **Thinking is enabled and subagents are disabled for IM channel runs by default** `[codebase]` — `DEFAULT_RUN_CONTEXT` sets `thinking_enabled: True, subagent_enabled: False`. These differ from the web UI defaults (subagents may be on there). An IM user who asks a research task will get a thinking-enabled response without subagent delegation unless the channel config explicitly overrides. Evidence: `app/channels/manager.py:L26-L30` — `DEFAULT_RUN_CONTEXT: dict[str, Any] = {"thinking_enabled": True, "is_plan_mode": False, "subagent_enabled": False}`

- **IM artifact delivery restricts to `/mnt/user-data/outputs/` — uploads and workspace files are blocked** `[codebase]` — `_resolve_attachments` rejects any artifact path not under `/mnt/user-data/outputs/` with a warning, preventing workspace files or uploads from being leaked through IM channels even if the agent calls `present_files` with those paths. A double path-traversal check using `resolve().relative_to()` guards against symlinks inside the outputs dir. Evidence: `app/channels/manager.py:L285-L295` — `if not virtual_path.startswith(_OUTPUTS_VIRTUAL_PREFIX): logger.warning("[Manager] rejected non-outputs artifact path: %s", virtual_path)`

- **Streaming update rate is throttled at 350ms minimum for non-final messages; final message always bypasses the limit** `[codebase]` — without this, rapid LangGraph SSE events would flood Feishu's card update API and trigger rate limiting. The 350ms floor was chosen to give perceptible streaming without API errors. Evidence: `app/channels/manager.py:L31,L634` — `STREAM_UPDATE_MIN_INTERVAL_SECONDS = 0.35` … `if last_published_text and now - last_publish_at < STREAM_UPDATE_MIN_INTERVAL_SECONDS: continue`

- **LangGraph recursion limit is set to 100 in IM channel runs vs framework default (~25)** `[codebase]` — IM users issue open-ended research requests that may require deep tool call chains. At the framework default, these frequently hit the recursion limit and terminate early. Evidence: `app/channels/manager.py:L25` — `DEFAULT_RUN_CONFIG: dict[str, Any] = {"recursion_limit": 100}`

### Trade-offs

| Decision point | Choose A when... | Choose B when... | This codebase chose | Structural consequence |
|---|---|---|---|---|
| Custom agent routing | You need separate LangGraph graphs per agent | You can differentiate via context injection into a single graph | `lead_agent` + `agent_name` context | All IM runs appear as `lead_agent` in monitoring; custom agent runs are opaque at the LangGraph level |
| Channel thread persistence | High concurrency, multi-process deployment | Single-process, low concurrency | JSON file + atomic `replace()` | One corrupt file loses all channel-to-thread mappings; not safe if multiple processes write concurrently |
| Streaming capability | All platforms support SSE | Some platforms are webhook-only | Per-channel capability table (Feishu: streaming, Slack/Telegram: `runs.wait`) | Adding a new platform requires updating `CHANNEL_CAPABILITIES` — no runtime config, code-level constant |

### Non-obvious behaviors

- **`_extract_response_text` stops scanning at the last human message** `[technology]` — when walking backwards through a multi-turn thread's message list, the extractor stops as soon as it hits a human message, ensuring responses from previous conversation turns cannot be re-sent. Evidence: `app/channels/manager.py:L96-L99` — `if msg_type == "human": break`

- **Streaming text accumulation handles both delta and cumulative event formats transparently** `[technology]` — `_merge_stream_text` uses heuristics: if the new chunk starts with existing text, it's a cumulative snapshot (replace); if existing ends with the chunk, it's a repeated delta (keep existing); otherwise concatenate. Callers never need to know which format LangGraph is emitting. Evidence: `app/channels/manager.py:L156-L166` — `if chunk.startswith(existing): return chunk` / `if existing.endswith(chunk): return existing`

- **`/bootstrap` command is a special case that routes to `_handle_chat`, not `_handle_command`** `[codebase]` — unlike `/new`, `/status`, `/models`, which are handled locally, `/bootstrap` dispatches an actual agent run with `is_bootstrap: True` context injected. A developer reading the command handler expecting local execution will miss that bootstrap triggers a full LLM call. Evidence: `app/channels/manager.py:L699-L705` — `chat_msg = _dc_replace(msg, text=chat_text, msg_type=InboundMessageType.CHAT); await self._handle_chat(chat_msg, extra_context={"is_bootstrap": True})`

### Structural anti-patterns

- **`ChannelStore` uses `threading.Lock` while `get_thread_id` reads `_data` without the lock** — `set_thread_id` holds the lock during write + save, but `get_thread_id` reads `self._data` directly. In asyncio (single-threaded), this works today; any future path that calls `set_thread_id` from a thread pool (`run_in_executor`) creates a real read-after-write race. Correct structure: use `asyncio.Lock` and make all methods async. Evidence: `app/channels/store.py:L82-L85` — `def get_thread_id(self, ...) -> str | None: entry = self._data.get(self._key(...))` (no lock acquired)

---

## Gateway API Layer

### Decisions

- **Thread listing uses two-source architecture: Store primary, checkpointer supplemental with lazy migration** `[codebase]` — threads created directly by LangGraph Server bypass the Store and only appear in listing after a Phase 2 checkpointer scan. Callers that only query the Store will silently miss those threads until the migration runs. Evidence: `app/gateway/routers/threads.py:L317-L419` — `async for checkpoint_tuple in checkpointer.alist(None): ... if store is not None: ... await _store_upsert(store, thread_id, ...)`

- **Active content MIME types (HTML/XHTML/SVG) are always forced to attachment download, ignoring the `?download=true` flag** `[codebase]` — the download flag is ineffective for these types: the endpoint overrides it. A caller expecting to control inline rendering of agent-generated HTML via the query param will find no difference in behavior. Evidence: `app/gateway/routers/artifacts.py:L170-L173` — `if mime_type in ACTIVE_CONTENT_MIME_TYPES: return FileResponse(... headers=_build_attachment_headers(...))`

- **CORS is delegated entirely to nginx — no FastAPI `CORSMiddleware` is installed** `[codebase]` — the gateway cannot be reached directly by browser clients without nginx. Developers running the Gateway standalone will get CORS errors with no visible explanation in the code. Evidence: `app/gateway/app.py:L166` — `# CORS is handled by nginx - no need for FastAPI middleware`

- **Sub-graph checkpoints are filtered from thread search by asserting `checkpoint_ns == ""`** `[technology]` — LangGraph writes sub-graph checkpoints with a non-empty `checkpoint_ns`. Without this filter, sub-graph states appear as phantom threads in the listing. Evidence: `app/gateway/routers/threads.py:L373` — `if cfg.get("configurable", {}).get("checkpoint_ns", ""): continue`

- **`upload_files` calls `sandbox_provider.acquire()` without a corresponding `release()`** `[codebase]` — the upload handler acquires the sandbox only to propagate files into non-local sandbox containers (`sandbox.update_file()`). Lifecycle management (release) remains SandboxMiddleware's responsibility; the upload handler is intentionally not the owner. A developer who adds release here would prematurely destroy agent sandboxes. Evidence: `app/gateway/routers/uploads.py:L72-L74` — `sandbox_id = sandbox_provider.acquire(thread_id); sandbox = sandbox_provider.get(sandbox_id)` (no release call)

### Trade-offs

| Decision point | Choose A when... | Choose B when... | This codebase chose | Structural consequence |
|---|---|---|---|---|
| Thread index source of truth | You need single fast lookup | You need to discover threads from all sources (LangGraph Server + Gateway) | Store + checkpointer lazy migration | First search after LangGraph Server creates a thread hits slow checkpointer scan; Store converges to full index over time without migration job |
| File type detection for text response | Extensions always present and reliable | Files may have wrong/missing extensions | Hybrid: `mimetypes.guess_type` + 8KB content null-byte sniff | Text files without extensions served as plain text; binary detection costs an 8KB disk read per unknown-extension file |
| Concurrent run strategy default | Maximum throughput matters | Preventing state corruption matters | `reject` (`multitask_strategy` default) | Concurrent runs on same thread return 409 — all callers must handle or retry |

### Non-obvious behaviors

- **Skill archive contents are transparently served from within ZIP files** `[codebase]` — a URL of the form `xxx.skill/SKILL.md` triggers in-request ZIP extraction. The file is never extracted to disk; callers can preview skill archive contents without installing them. Evidence: `app/gateway/routers/artifacts.py:L117-L153` — `if ".skill/" in path: ... content = _extract_file_from_skill_archive(actual_skill_path, internal_path)`

- **Uploaded files are chmod'd world-writable (0o666) for non-local sandboxes** `[codebase]` — when `sandbox_id != "local"`, `_make_file_sandbox_writable` widens permissions. Without this, the sandbox container user (different UID from gateway process) cannot write to mounted upload paths. Evidence: `app/gateway/routers/uploads.py:L38-L53` — `writable_mode = stat.S_IMODE(...) | S_IWUSR | S_IWGRP | S_IWOTH`

- **`update_thread_state` immediately syncs `title` changes back to the Store** `[codebase]` — when `title` is in `body.values`, the handler calls `_store_upsert` so subsequent `search_threads` responses reflect the rename without delay. Any other state field updated here does NOT get synced to the Store. Evidence: `app/gateway/routers/threads.py:L622-L626` — `if store is not None and body.values and "title" in body.values: await _store_upsert(store, thread_id, values={"title": body.values["title"]})`

- **Store search failure silently degrades to checkpointer-only results** `[codebase]` — if `store.asearch()` raises, the exception is caught, logged at WARNING, and the search continues with an empty `merged` dict. The caller receives checkpointer-discovered threads only, with no error response. Evidence: `app/gateway/routers/threads.py:L347-L350` — `except Exception: logger.warning("Store search failed — falling back to checkpointer only", exc_info=True); items = []`

### Structural anti-patterns

- **`upload_files` reads entire file content into memory before writing** — for large uploads (100MB+ documents), this exhausts gateway process memory and blocks the event loop during the `await file.read()` call. Correct structure: stream `file.file` directly to disk using `aiofiles` or a chunked write loop. Evidence: `app/gateway/routers/uploads.py:L87-L88` — `content = await file.read(); file_path.write_bytes(content)`

---
