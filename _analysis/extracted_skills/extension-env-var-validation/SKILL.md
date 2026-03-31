---
name: extension-env-var-validation
description: "Prevents silently misconfigured extensions when env var placeholders in plugin config resolve to empty string at load time. TRIGGER when: adding env var references to MCP server or plugin configs, designing config resolution for user-supplied extension manifests, choosing between fail-fast and silent-fallback for missing environment variables in optional integrations."
---

# Extension Env Var Validation
*Prevents extensions from appearing configured while silently failing on first use due to empty-string env var fallbacks.*

## Key decisions

1. Validate env var references in extension configs at load time, not at first call. Without this, a missing `$API_KEY` placeholder resolves to `""`, the MCP server starts successfully, serves its tool schema to the agent, and fails only during a live user session when the agent first invokes the tool — startup tests never reveal it.

2. Distinguish required env vars (fail at startup) from optional ones (safe to omit with empty string). Without this, you must choose between blocking startup for any missing key — breaking optional integrations — or silently misconfiguring required ones.

3. Use fail-fast semantics for required keys: raise at config resolution time with the variable name, not a generic connection error. Without this, a misconfigured API key produces a cryptic 401 or `ConnectionRefusedError` deep in an agent tool call, with no signal pointing back to the config.

## Anti-patterns

- **What**: Replace any unresolved `$VAR` placeholder with `""` as a safe universal fallback
- **Why**: The empty string propagates as an API key or connection string; passes all type checks; is only rejected by the external service at call time
- **Symptom**: Extension tool appears in the agent's tool list; every invocation returns 401 or connection error; appears hours after deployment when the agent first uses the tool

- **What**: Validate env vars only after a tool call failure
- **Why**: The validation is reactive — the extension is already serving requests with an empty key
- **Symptom**: First tool invocation by the agent fails; retries accumulate; root cause traced back to a missing env var that was set in staging but not production

## Structural template

```
# Config resolution for extension env vars

REQUIRED_ENV_KEYS = {"api_key", "token", "secret"}

def resolve_extension_env(config: dict) -> dict:
    resolved = {}
    missing_required = []

    for key, value in config.items():
        if not is_env_placeholder(value):
            resolved[key] = value
            continue

        var_name = extract_var_name(value)  # "$FOO_KEY" → "FOO_KEY"
        env_value = os.environ.get(var_name)

        if env_value is None:
            if key in REQUIRED_ENV_KEYS:
                missing_required.append(var_name)
            else:
                resolved[key] = ""  # safe: optional key
        else:
            resolved[key] = env_value

    if missing_required:
        raise ConfigurationError(
            f"Extension requires env vars that are not set: {missing_required}"
        )

    return resolved

# Validate at load time — before serving tool schemas
extensions = load_extensions_config()
for ext_name, ext_config in extensions.items():
    try:
        resolve_extension_env(ext_config.env)
    except ConfigurationError as e:
        logger.warning("Extension %s disabled: %s", ext_name, e)
        mark_extension_unavailable(ext_name, reason=str(e))
        # do not raise — allow other extensions to load
```
