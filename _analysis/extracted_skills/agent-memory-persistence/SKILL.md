---
name: agent-memory-persistence
description: "Ensures agent memory file writes are non-blocking and crash-safe in production. TRIGGER when: adding persistent memory to a multi-turn agent, memory updates slowing agent response times, agent memory files corrupted after process crash or restart, debugging partial writes or JSON parse errors in memory storage."
---

## Key Decisions

1. **Run memory persistence in a background thread with a debounce queue, not inline after each message.** Memory extraction requires one or more LLM calls; running them in-band blocks the user-facing response until they complete. A debounce window (e.g., 30 seconds) batches rapid back-and-forth turns into a single extraction call and decouples memory persistence latency from response latency entirely.

2. **Use atomic writes (write to a temp file, then rename) when persisting memory files.** If the process crashes mid-write to the target file, the file is left partially written; subsequent reads will fail to parse it. Renaming a fully written temp file is atomic on most operating systems — readers always see either the old complete file or the new complete file, never an intermediate state.

## Anti-patterns

- **What**: Calling memory extract-and-store inline as part of the main response pipeline.
  **Why**: Memory extraction requires LLM calls; running them in-band blocks the response until they complete.
  **Symptom**: Response latency spikes with memory enabled; users experience degraded turn-around time proportional to memory extraction time on every exchange.

- **What**: Writing directly to the memory file path without an intermediate temp file.
  **Why**: Process termination (SIGKILL, OOM, crash) can occur at any point during a write, leaving the file truncated or with invalid JSON.
  **Symptom**: Agent fails to start or returns empty memory after any unclean shutdown; the file must be manually deleted to recover.

## Structural Template

```
# Initialization
debounce_queue = Queue()
background_thread = Thread(target=memory_flush_loop, daemon=True)
background_thread.start()


# Called after each user message — non-blocking
function enqueue_memory_update(conversation_turn):
    debounce_queue.put(conversation_turn)


# Runs in background thread
function memory_flush_loop():
    while True:
        turn = debounce_queue.get()
        wait_for_quiet_period(seconds=DEBOUNCE_WINDOW)

        # Drain any turns queued during the wait window
        batch = [turn] + drain_queue(debounce_queue)

        updated_facts = extract_facts(batch)
        atomic_write(MEMORY_PATH, updated_facts)


function atomic_write(path, data):
    tmp = path + ".tmp"
    write_file(tmp, serialize(data))
    rename(tmp, path)  # atomic on POSIX; readers never see partial content
```
