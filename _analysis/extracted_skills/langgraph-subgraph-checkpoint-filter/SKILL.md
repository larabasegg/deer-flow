---
name: langgraph-subgraph-checkpoint-filter
description: "Prevents phantom thread entries when listing threads from the LangGraph checkpointer in deployments with hierarchical agent graphs. TRIGGER when: implementing thread listing from a LangGraph checkpointer, building a thread index or search feature over LangGraph state, designing a system that uses sub-graphs or parallel node execution within a parent LangGraph graph."
---

# LangGraph Sub-graph Checkpoint Filtering
*Prevents phantom threads in listings caused by LangGraph writing sub-graph checkpoints with non-empty `checkpoint_ns` values.*

## Key decisions

1. Filter checkpointer entries by `checkpoint_ns == ""` when building a thread listing. Without this, every sub-graph invocation writes its own checkpoint with a non-empty namespace, and these appear as separate threads in the listing — a graph with 5 parallel sub-agents generates 5 phantom entries per run alongside the one real thread.

2. Apply the `checkpoint_ns` filter at the checkpointer query level, before merging with any secondary thread store. Without this, phantom threads accumulate in secondary stores during lazy migration and are never cleaned up, growing proportionally to production traffic.

3. Treat the empty-namespace filter as a structural invariant rather than post-processing. Without this, any future code path that adds a second listing route will silently reintroduce phantom threads, and the regression may not surface in tests that use flat single-graph topologies.

## Anti-patterns

- **What**: List all entries from `checkpointer.alist()` without filtering by namespace
- **Why**: The LangGraph API makes all checkpoints structurally identical; sub-graph checkpoints have the same fields as top-level thread checkpoints but represent internal implementation state, not user-visible threads
- **Symptom**: Thread listing returns N × M entries for N user threads with M sub-graphs each; users see phantom conversations they never started; appears only in production workloads with hierarchical or parallel agent graphs — test graphs are flat

- **What**: Filter phantom threads at render time (UI layer) rather than at the data layer
- **Why**: The phantom threads still accumulate in stores and affect count-based queries, pagination, and search
- **Symptom**: Thread count is inflated; search returns irrelevant results from sub-graph state; pagination skips real threads

## Structural template

```
# Thread listing from LangGraph checkpointer

async def list_threads_from_checkpointer(
    checkpointer,
) -> list[dict]:
    threads = []

    async for checkpoint_tuple in checkpointer.alist(None):
        cfg = checkpoint_tuple.config.get("configurable", {})

        # Filter: skip sub-graph checkpoints.
        # LangGraph writes sub-graphs with non-empty checkpoint_ns.
        # Only top-level threads have checkpoint_ns == "".
        if cfg.get("checkpoint_ns", ""):
            continue

        thread_id = cfg.get("thread_id")
        if not thread_id:
            continue

        threads.append(
            {
                "thread_id": thread_id,
                "created_at": checkpoint_tuple.checkpoint.get("ts"),
                "metadata": checkpoint_tuple.metadata,
            }
        )

    return threads


# When merging into a secondary store, apply filter first.
# Phantom threads never reach the store — no cleanup job needed.
async def sync_checkpointer_to_store(checkpointer, store):
    for thread in await list_threads_from_checkpointer(checkpointer):
        await store.upsert(thread["thread_id"], thread)
```
