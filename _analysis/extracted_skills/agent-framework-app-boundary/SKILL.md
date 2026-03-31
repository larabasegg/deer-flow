---
name: agent-framework-app-boundary
description: "Keeps agent framework code decoupled from application layer code and ensures cross-process config changes propagate without restarts. TRIGGER when: packaging an agent framework for embedded or library use, deploying an agent framework and API gateway as separate processes, debugging stale tool configs that don't update after config edits, ensuring config changes in one process are visible to another without restart."
---

## Key Decisions

1. **Enforce a strict import boundary between the agent framework layer and the application layer, validated in CI.** Without an enforced boundary, framework code that imports application code becomes entangled with the HTTP gateway, deployment config, and database access — making it impossible to embed the framework in other contexts or publish it as a reusable library. A CI test that fails on cross-layer imports is the only enforcement that survives refactoring.

2. **Load external tool configurations lazily at first use with file modification time (mtime) cache invalidation, not eagerly at startup.** Eager loading fails when an external service is not yet running at agent startup. In multi-process deployments, config changes written by one process (e.g., an API server) must be visible to another process (e.g., the agent server) between requests; mtime-based invalidation detects file changes without a coordinated restart.

3. **Cache parsed application config but auto-reload when the config file's mtime increases.** When the agent framework and its API gateway run as separate processes reading the same config file, changes written by one process are never seen by the other without a reload mechanism. Mtime-based reload makes runtime config changes immediately effective without requiring process restarts or inter-process signaling.

## Anti-patterns

- **What**: Agent framework code that imports from the application or gateway layer.
  **Why**: Cross-layer imports couple the framework to the deployment topology, preventing embedded use and library packaging.
  **Symptom**: Attempting to run the framework outside the main app (e.g., in tests, as a library) fails with import errors referencing HTTP, database, or gateway modules.

- **What**: Loading all external tool configurations at agent startup.
  **Why**: Startup-time loading fails if the tool's backing service is not yet available when the agent initializes.
  **Symptom**: Agent fails to start whenever an external tool service is down, even if no tool calls will be made; config files edited while the agent runs have no effect until restart.

- **What**: Caching config in-process with no invalidation mechanism in multi-process deployments.
  **Why**: Two processes both reading the same config file will diverge as soon as one writes a change; the other process sees stale config indefinitely.
  **Symptom**: Config changes applied via the API take effect immediately in the API process but never propagate to the agent process; requires manual restart to sync.

## Structural Template

```
# CI boundary test — fails if framework imports application layer
function test_framework_does_not_import_app_layer():
    framework_sources = find_source_files("framework/")
    for source in framework_sources:
        imports = extract_imports(parse(source))
        for imp in imports:
            assert not is_app_layer_import(imp),
                f"{source}: framework must not import from app layer"


# Config with mtime-based reload (multi-process safe)
class AppConfig:
    def __init__(self, path):
        self._path = path
        self._mtime = None
        self._config = None

    def get(self, key):
        current_mtime = file_mtime(self._path)
        if current_mtime != self._mtime:
            self._config = load_config(self._path)
            self._mtime = current_mtime
        return self._config[key]


# Lazy tool registry with mtime-based invalidation
class LazyToolRegistry:
    def __init__(self, config_path):
        self._config_path = config_path
        self._mtime = None
        self._tools = None

    def get_tool(self, name):
        current_mtime = file_mtime(self._config_path)
        if current_mtime != self._mtime:
            self._tools = load_tool_config(self._config_path)
            self._mtime = current_mtime
        return self._tools[name]
```
