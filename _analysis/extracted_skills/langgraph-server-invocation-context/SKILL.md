---
name: langgraph-server-invocation-context
description: "LangGraph Server invocation context: failure modes when reading thread_id, user_id, or other configurable fields inside a LangGraph agent deployed behind LangGraph Server. TRIGGER when: writing middleware or tools that access thread_id or user_id, deploying a LangGraph agent via LangGraph Server (not direct Python invocation), debugging silent failures where context reads return None in server mode but work locally."
---

# LangGraph Server Invocation Context
*What to expect when reading invocation context inside a LangGraph agent deployed via LangGraph Server — failure modes in the order you will encounter them.*

## Who this is for
Teams deploying LangGraph agents via LangGraph Server. These failure modes appear as soon as code that works in direct Python invocation is deployed behind the server.

## Failure mode 1: context field returns None in server mode
**Symptom:** `thread_id`, `user_id`, or other invocation context fields read as `None` (or raise `AttributeError`) when the agent runs via LangGraph Server, but return correct values during direct Python invocation or local testing.

**Mechanism:** LangGraph has two locations for invocation context depending on how the agent is invoked. When invoked directly in Python, context lives in `runtime.context`. When invoked via LangGraph Server (HTTP), context lives in `config.configurable`. Code that reads only one source silently gets `None` in the other deployment mode.

**Fix:** Always implement a two-source fallback for any context field:
```python
thread_id = (
    runtime.context.get("thread_id")
    or config.configurable.get("thread_id")
)
```
Apply this pattern in every location that reads invocation context: middlewares, tools, and any lazily-initialized components.

**Evidence:** Open-source AI agent project — March 2026

## Failure mode 2: same bug re-introduced independently in multiple components
**Symptom:** After fixing the context read in one location, identical `None` returns appear in other components (middlewares, tools, lazy init) that each independently assumed a single context source.

**Mechanism:** The two-source fallback must be applied at every callsite. There is no central context accessor that all components use — each reads directly from `runtime.context` or `config.configurable`. A fix in one component does not propagate to others.

**Fix:** Extract the fallback into a shared helper and use it everywhere. Audit all middlewares and tools for bare `runtime.context.get(...)` calls.

**Evidence:** Open-source AI agent project — March 2026 (4 independent components affected in the same week)

## Failure mode 3: lazy initialization components read context too early or from the wrong source
**Symptom:** A component that initializes on first use (e.g., a sandbox, workspace, or session object) fails silently or initializes with a `None` thread_id, causing downstream errors that appear unrelated to context reading.

**Mechanism:** Lazy init code often captures context at construction time. If the component is constructed before context is available, or reads only `runtime.context` while running behind LangGraph Server, it captures `None` and never retries.

**Fix:** Defer context reads to the first actual use (not construction), and always use the two-source fallback. Validate that context fields are non-None before proceeding with initialization.

**Evidence:** Open-source AI agent project — March 2026

## The underlying pattern
LangGraph's invocation context is split across two locations (`runtime.context` and `config.configurable`) depending on the deployment mode, and there is no automatic unification. Any code path that reads context must implement a two-source fallback — the omission is silent, returns `None`, and will not surface in local testing where only one invocation mode is used. The pattern recurs independently in every component that touches context, so a shared accessor is the only durable fix.
