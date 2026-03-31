# Cluster D: Data and Adaptation

## SECTION 8: LEARNING AND ADAPTATION

**Does the system learn from past interactions?**

Partial — memory extraction mechanism stores user facts and context summaries across sessions, which influences future agent behavior. This is not learning in the ML sense, but behavioral adaptation via persistent context.

- Memory is updated after each conversation via LLM-based fact extraction — evidence: `agents/memory/updater.py:269-333`
- Extracted facts are injected into system prompt in future sessions — evidence: `agents/lead_agent/prompt.py:351-379`
- The agent adapts responses based on stored user context (work context, personal context, preferences as facts)

**Mechanism:**

| **Mechanism** | **Implementation** | **Evidence** |
|---|---|---|
| Fact extraction from conversations | `MemoryUpdater.update_memory()` calls LLM → JSON → `_apply_updates()` | `agents/memory/updater.py:269-412` |
| User context summarization | `workContext`, `personalContext`, `topOfMind` summaries updated when LLM flags `shouldUpdate: true` | `updater.py:354-362` |
| History summarization | `recentMonths`, `earlierContext`, `longTermBackground` updated similarly | `updater.py:364-371` |
| Context injection | Memory injected as `<memory>` block in system prompt each session | `agents/lead_agent/prompt.py:351-379` |

**Explicit reinforcement (thumbs up/down, reward signals):** No implementation — no feedback capture, no rating UI, no reward signal

**Versioning of learned strategies:** No implementation — memory.json has `"version": "1.0"` schema version but no behavioral strategy versioning; no rollback mechanism beyond restoring `memory.json` from backup

**Scope of adaptation:**
- Global by default — `memory.json` shared across all threads and sessions
- Per-agent scope available: `agent_name` parameter routes to separate memory file — evidence: `agents/memory/storage.py:76-86`
- Not scoped per-user or per-task-type (no user identity model)

**Online vs offline learning:**
- Offline — memory updates run in background thread via debounced queue (30s default) after conversation ends
- No model weight updates; adaptation is prompt-level only

**Safeguards against incorrect patterns:**
- Confidence threshold: facts below `fact_confidence_threshold` (default: 0.7) are discarded — evidence: `updater.py:383-384`
- Max facts limit: sorted by confidence, lowest-confidence facts pruned when `max_facts` (100) exceeded — evidence: `updater.py:404-410`
- Upload mention filtering: sentences about session-scoped uploaded files stripped before saving — evidence: `updater.py:319-323`
- No human review gate — all LLM-extracted facts passing confidence threshold are auto-applied
