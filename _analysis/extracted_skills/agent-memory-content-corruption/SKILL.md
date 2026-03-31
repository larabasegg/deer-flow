---
name: agent-memory-content-corruption
description: "Prevents agent long-term memory from degrading due to session-scoped references, low-confidence extractions, and cross-agent or cross-user contamination. TRIGGER when: building persistent memory for a multi-turn agent, memory retrieved in future sessions contains stale or invalid references, agent performance degrades as memory grows, deploying an agent in a multi-user or multi-agent environment."
---

## Key Decisions

1. **Strip session-scoped references before persisting facts.** Files uploaded in a session, temporary URLs, or other session-bound entities will not exist in future sessions. If the agent stores "user uploaded X" as a long-term fact, it will hallucinate or fail when that reference is retrieved later. Apply a filter (e.g., regex matching upload/session event patterns) before any memory persist call.

2. **Apply a confidence threshold before storing LLM-extracted facts, and prune the lowest-confidence facts when the store reaches capacity.** LLM extraction is noisy — low-confidence extractions pollute long-term memory and degrade future responses when injected into system prompts. Discarding facts below a threshold (e.g., 0.7) and pruning the weakest facts at capacity prevents quality degradation over time.

3. **Scope memory by agent identity, not just by thread or session.** In systems with multiple specialized agents, one agent's memory can pollute another's retrieval context. Route facts to stores keyed by agent identity so that different agent roles do not share a memory namespace. The same principle applies at the user level in multi-user deployments — shared memory across users causes private facts to leak between users.

## Anti-patterns

- **What**: Storing session-scoped references (uploaded file names, temporary URLs, session identifiers) as durable long-term facts.
  **Why**: LLM memory extraction does not distinguish between persistent facts and session events; it extracts both indiscriminately.
  **Symptom**: Agent attempts to access files or URLs in new sessions that no longer exist, producing hallucinated content or unexplained tool errors on every fresh conversation.

- **What**: Using a single global memory store shared across all users in a multi-user deployment.
  **Why**: LLM-extracted facts contain personal context (work details, preferences, identity) that will be injected into every user's system prompt from the shared store.
  **Symptom**: Users receive memory-augmented responses containing another user's personal context; a privacy breach that requires a full store wipe to resolve.

- **What**: Storing all LLM-extracted facts without quality filtering.
  **Why**: LLM extraction over ambiguous or low-signal turns produces weak or incorrect facts that accumulate over time.
  **Symptom**: Agent memory grows with contradictory or irrelevant facts; response quality degrades as the injected memory context becomes noise-dominated.

## Structural Template

```
function extract_and_store_facts(conversation_turn, agent_id, user_id):
    candidates = llm_extract_facts(conversation_turn)

    # Filter 1: Confidence gate
    candidates = [f for f in candidates if f.confidence >= CONFIDENCE_THRESHOLD]

    # Filter 2: Strip session-scoped references
    candidates = [f for f in candidates if not is_session_scoped(f.content)]

    # Route to correct store — never use a global store
    store_key = compose_key(agent_id, user_id)
    store = get_store(store_key)

    store.add(candidates)

    # Capacity management: prune lowest-confidence facts
    if len(store) > MAX_FACTS:
        store.prune(n=len(store) - MAX_FACTS, strategy="lowest_confidence")


function is_session_scoped(text):
    # Match upload events, temp URLs, session tokens, etc.
    return matches_session_reference_patterns(text)
```
