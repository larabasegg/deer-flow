---
name: langgraph-client-thread-persistence
description: "Prevents multi-turn conversation amnesia when a LangGraph client accepts thread_id but has no checkpointer configured. TRIGGER when: implementing a LangGraph client that accepts thread_id, building a multi-turn conversation interface on top of a LangGraph graph, choosing whether to configure a checkpointer for a stateful agent client."
---

# LangGraph Client Thread Persistence
*Prevents silent statelessness when thread_id is accepted by a client but no checkpointer is configured.*

## Key decisions

1. A `thread_id` parameter on a LangGraph client controls file isolation only — it does not enable conversation persistence unless a checkpointer is explicitly configured. Without a checkpointer, every `stream()` call starts a fresh stateless conversation regardless of `thread_id`, and no error is raised; the response looks valid.

2. Configure a checkpointer before exposing multi-turn interfaces. Without this, a client that accepts `thread_id` from callers silently delivers amnesia: the first turn always succeeds, subsequent turns have no memory of prior context, and callers have no way to distinguish this from a functioning multi-turn session.

3. Log or warn at initialization time if the client is stateless. Without this, a missing checkpointer configuration is invisible in the call path — the only signal is user-reported context loss, which appears only in production multi-turn sessions that demos and smoke tests never exercise.

## Anti-patterns

- **What**: Pass `thread_id` to client calls expecting conversation history to carry across calls
- **Why**: Without a checkpointer, the graph loads no state; thread_id only maps to a filesystem directory for upload and output file isolation
- **Symptom**: Users report the agent forgot context from the previous message; single-turn unit tests pass; multi-turn integration tests or manual QA catch it only if explicitly testing follow-up questions

- **What**: Treat a successful first-turn response as proof that persistence is working
- **Why**: The first call always succeeds (there is nothing to load from history); failure only appears on the second call when the agent ignores prior context
- **Symptom**: Demos using two-question sequences pass; real user sessions with back-references ("as I mentioned earlier...") fail; discovered in production, not staging

## Structural template

```
# LangGraph client initialization

# WITHOUT persistence (stateless — each call is independent)
# thread_id is file isolation only — not conversation memory
client = AgentClient(graph=graph)

# WITH persistence (stateful multi-turn)
checkpointer = SqliteSaver.from_conn_string("sessions.db")
# or: MemorySaver(), PostgresSaver, RedisSaver, etc.
client = AgentClient(graph=graph, checkpointer=checkpointer)


class AgentClient:
    def __init__(self, graph, checkpointer=None):
        self._graph = graph
        self._checkpointer = checkpointer

        if checkpointer is None:
            logger.warning(
                "No checkpointer configured — each call is stateless. "
                "thread_id is used for file isolation only."
            )

    def stream(self, state, thread_id, **kwargs):
        run_config = {
            "configurable": {
                "thread_id": thread_id,  # always: controls file paths
            }
        }

        if self._checkpointer is None:
            # stateless: no history loaded or saved between calls
            return self._graph.stream(
                state, config=run_config, stream_mode="values"
            )
        else:
            # stateful: checkpointer loads and saves history per thread_id
            return self._graph.stream(
                state,
                config=run_config,
                checkpointer=self._checkpointer,
                stream_mode="values",
            )
```
