---
name: subagent-concurrency-execution
description: "Prevents deadlocks, silent work loss, and async failures when running concurrent subagents in a thread pool. TRIGGER when: building a multi-agent system with concurrent subagent execution, subagent tasks silently disappearing or never executing, adding async tool support to thread-pool-based agent executors, limiting the number of concurrent subagent invocations."
---

## Key Decisions

1. **Enforce subagent concurrency limits at the application layer by truncating excess tool invocations from the model response, not at the API level.** LLM APIs do not enforce per-response tool call count limits. If a model emits more task-dispatch calls than available executor slots, the excess calls must be dropped explicitly in application code; otherwise they will queue indefinitely or silently fail to execute with no error surfaced to the caller.

2. **Use separate thread pools for scheduling and execution to prevent deadlock.** If a single pool handles both the scheduler (which calls `future.result()` with a timeout) and the executor (which runs the subagent), the scheduler thread blocks a pool worker while waiting — leaving no threads available for the execution task. Separate pools guarantee the scheduler can always wait without consuming an executor slot.

3. **Each subagent worker thread that invokes async tools must run its own event loop.** Thread pool workers have no event loop by default. Attempting to await async tool calls (e.g., async network I/O) from a thread pool worker without initializing an event loop will raise a runtime error. Starting a fresh event loop per worker ensures async tools function correctly within the executor.

## Anti-patterns

- **What**: Relying on the LLM API or the agent framework to enforce a cap on how many subagent dispatches appear in one model response.
  **Why**: LLMs do not observe executor capacity when generating responses; they emit as many tool calls as their planning warrants.
  **Symptom**: Model emits 10 subagent dispatches when only 3 slots exist; excess work is silently dropped or causes thread pool exhaustion with no error message.

- **What**: Using a single thread pool where the scheduler blocks waiting for the executor.
  **Why**: `future.result()` in the scheduler holds a pool worker; with a single shared pool, the execution task has no thread to run on.
  **Symptom**: All subagent tasks submitted simultaneously hang indefinitely; the system appears stalled with no deadlock warning.

- **What**: Calling async tool methods directly from a thread pool worker without initializing an event loop.
  **Why**: Thread pool workers inherit no event loop from the parent process.
  **Symptom**: The first subagent to invoke an async tool fails immediately with a "no event loop" runtime error; subsequent workers hit the same failure with no useful stack context.

## Structural Template

```
SCHEDULER_POOL_SIZE = 3
EXECUTION_POOL_SIZE = 3
MAX_CONCURRENT_TASKS = EXECUTION_POOL_SIZE

scheduler_pool = ThreadPool(SCHEDULER_POOL_SIZE)
execution_pool = ThreadPool(EXECUTION_POOL_SIZE)


function dispatch_subagents(model_response):
    tool_calls = extract_task_calls(model_response)

    # Enforce limit at application layer — truncate, do not queue
    tool_calls = tool_calls[:MAX_CONCURRENT_TASKS]

    futures = [execution_pool.submit(run_subagent, call)
               for call in tool_calls]

    return scheduler_pool.submit(collect_results, futures)


function run_subagent(task):
    # Each worker initializes its own event loop for async tools
    return run_event_loop(execute_task_async(task))


function collect_results(futures):
    return [f.result(timeout=SUBAGENT_TIMEOUT) for f in futures]
```
