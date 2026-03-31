---
name: docker-compose-env-precedence
description: "Docker Compose environment variable precedence: failure modes when mixing the environment section and env_file for the same variable. TRIGGER when: writing docker-compose.yml with both environment and env_file sections, using \${VAR:-default} syntax in an environment section, debugging why .env overrides are silently ignored at docker compose up."
---

# Docker Compose Environment Variable Precedence
*What to expect when mixing the `environment` section and `env_file` in Docker Compose — failure modes in the order you will encounter them.*

## Who this is for
Teams configuring Docker Compose services with both an `environment` section and an `env_file`. The failure mode appears when users expect `.env` to override a variable that also appears in `environment`.

## Failure mode 1: .env override silently ignored
**Symptom:** A variable is set in `.env` (or `env_file`) but the service always uses the default value. Changing the value in `.env` has no effect. No error is shown.

**Mechanism:** Docker Compose evaluates the `environment` section using the **host shell environment** at `docker compose up` time — before reading `env_file`. The `${VAR:-default}` syntax is a shell expansion resolved from the host shell, not from the `.env` file. If `VAR` is not set in the host shell, the expression resolves to `default` immediately, and the result is baked into the `environment` entry as a literal string. By the time `env_file` is loaded, the variable already has a fixed value from `environment`, and `env_file` entries for the same variable are ignored because `environment` takes precedence.

**Fix:** Choose one of:
1. **Remove the variable from `environment`** and rely solely on `env_file` / `.env`. The variable will be passed through from env_file without conflict.
2. **Document that the variable must be exported in the host shell** (not `.env`) before running `docker compose up`.
3. **Use ARG + ENV in Dockerfile** if a build-time default is needed, keeping runtime overrides via env_file only.

**Evidence:** Open-source AI agent project — March 2026

## Design insight 1: environment and env_file are not additive — environment always wins for the same key
**Behavior:** When the same variable appears in both `environment` and `env_file`, the `environment` value always takes effect. This is true even if `environment` holds a shell-expanded default (`${VAR:-false}`) that resolved to the fallback.

**Why it's non-obvious:** The mental model of "env_file sets defaults, environment overrides" is the reverse of what Docker Compose actually does. Developers expect `.env` to be the user-editable override layer, but `environment` in the Compose file has higher precedence than `env_file` for the same key.

**What callers must know:** Never put `${VAR:-default}` in the `environment` section if you intend `env_file` or `.env` to be the override mechanism for that variable. The expansion happens at compose-up time from the host shell, locking in the default before env_file is consulted.

**Evidence:** Open-source AI agent project — March 2026

## The underlying pattern
Docker Compose's variable resolution order has a non-obvious split: `environment` section values are shell-expanded from the host environment at startup, while `env_file` is loaded afterward and loses to `environment` for duplicate keys. Any pattern that uses `${VAR:-default}` in `environment` while expecting `.env` to override it will silently fail. The only safe approach is to keep a variable in one section only — either `environment` (for values that must come from the host shell) or `env_file` (for values users should configure in `.env`).
