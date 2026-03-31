---
name: langgraph-recursion-limit-research
description: "Prevents silent task truncation when a research or multi-step agent hits LangGraph's default recursion limit mid-task. TRIGGER when: configuring run parameters for a LangGraph agent that uses multi-step tool calls, setting up a research or planning agent with web search or code execution tools, tuning LangGraph runtime behavior for production workloads."
---

# LangGraph Recursion Limit for Research Agents
*Prevents silent task truncation when LangGraph's default recursion limit (~25) terminates multi-step research runs before completion.*

## Key decisions

1. Set the LangGraph recursion limit to 50–100 for agents that perform multi-step tool use (web search, code execution, document retrieval, sub-task delegation). Without this, the framework's default of ~25 steps causes deep research tasks to terminate early and return partial results — no exception is raised to the caller; the run completes normally with a truncated response.

2. Apply the recursion limit per-run in the run config, not as a global override. Without this, a high limit set globally masks infinite loops in lighter graph branches where a wrong exit condition causes a cycle — simple Q&A flows that should fail fast instead run to the full limit.

3. Catch `GraphRecursionError` and emit a user-visible signal or structured log entry. Without this, a truncated response is indistinguishable from a complete one at the API level — the client receives a valid-looking message with no indication that work was cut short.

## Anti-patterns

- **What**: Deploy a research agent with the framework's default recursion limit
- **Why**: Test queries resolve in 3–5 steps; real research with multi-source search, retry-on-failure, and sub-task delegation routinely requires 30–60 steps
- **Symptom**: Complex research tasks return partial answers; agent stops mid-reasoning with no error in logs; local tests with simple questions always pass; discovered only when users report incomplete answers

- **What**: Raise the limit globally to avoid per-run configuration overhead
- **Why**: A high global limit hides infinite loops in graph branches where a wrong exit condition triggers a cycle — they run silently to the ceiling instead of failing fast
- **Symptom**: Infinite loops in lightweight graph branches consume full token budget before stopping; simple Q&A flows take minutes instead of seconds

## Structural template

```
# Research agent run configuration

RESEARCH_RECURSION_LIMIT = 100   # vs LangGraph default ~25
QA_RECURSION_LIMIT = 15          # keep low to catch cycles fast

# Apply per-run — not globally
research_run_config = {
    "recursion_limit": RESEARCH_RECURSION_LIMIT,
    "configurable": {
        "thread_id": thread_id,
    },
}

# Detect and signal truncation
try:
    result = graph.invoke(state, config=research_run_config)
except GraphRecursionError:
    logger.error(
        "Recursion limit (%d) reached for thread %s — task may be incomplete",
        RESEARCH_RECURSION_LIMIT,
        thread_id,
    )
    return TruncatedResult(
        message="Research task exceeded step limit. Results may be incomplete.",
        partial=last_known_state,
    )

# Lightweight Q&A graph — separate low limit to catch loops fast
qa_run_config = {
    "recursion_limit": QA_RECURSION_LIMIT,
    "configurable": {"thread_id": thread_id},
}
```
