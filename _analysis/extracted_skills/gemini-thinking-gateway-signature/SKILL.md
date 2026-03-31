---
name: gemini-thinking-gateway-signature
description: "Gemini thinking models via OpenAI-compatible gateway: failure modes when routing Gemini thinking-enabled models through LangChain's ChatOpenAI and an OpenAI-compatible proxy. TRIGGER when: using Gemini thinking models via an OpenAI-compatible gateway, routing any model with thinking/reasoning content blocks through LangChain ChatOpenAI, debugging HTTP 400 errors on multi-turn conversations that include tool calls."
---

# Gemini Thinking Models via OpenAI-Compatible Gateway
*What to expect when routing Gemini thinking models through LangChain's ChatOpenAI and an OpenAI-compatible proxy — failure modes in the order you will encounter them.*

## Who this is for
Teams using Gemini thinking (extended reasoning) models via an OpenAI-compatible gateway, accessed through LangChain's `ChatOpenAI`. The failure mode appears as soon as the conversation has more than one turn that includes tool calls.

## Failure mode 1: HTTP 400 on the second turn of any conversation with tool calls
**Symptom:** Single-turn conversations and multi-turn text-only conversations succeed. As soon as a conversation reaches a second turn that includes tool calls (or follows a turn that had tool calls), the API returns HTTP 400. The error appears after the first tool call completes.

**Mechanism:** Gemini thinking models include a `thought_signature` field on thinking content blocks. Gemini validates that this field is echoed back verbatim on subsequent turns — it uses signatures to verify the reasoning chain has not been tampered with. `ChatOpenAI` serializes messages without understanding Gemini-specific fields and silently drops `thought_signature` during message conversion. When the next request reaches Gemini without the expected signatures, it rejects with HTTP 400.

**Fix:** Use a patched message adapter that re-injects signed thinking blocks before each request. The adapter must:
1. Inspect `additional_kwargs.thinking_blocks` or the message content list for thinking blocks carrying `thought_signature`
2. Re-insert those blocks into the outgoing message content before the request is sent
3. Preserve the `thought_signature` value exactly — any modification causes the same 400

**Evidence:** Open-source AI agent project — March 2026

## Design insight 1: thought_signature is a Gemini-internal validation field, not an optional annotation
**Behavior:** `thought_signature` on a thinking content block is not metadata — it is a cryptographic-style round-trip token that Gemini requires to be present and unmodified on follow-up turns. Omitting it is treated as a tampered or incomplete reasoning chain.

**Why it's non-obvious:** OpenAI's reasoning/thinking content blocks do not have an equivalent field. Any LangChain adapter built for OpenAI will not know to preserve it. Documentation for the gateway and LangChain does not surface this requirement.

**What callers must know:** You cannot rely on standard LangChain message serialization when using Gemini thinking models. A custom adapter at the serialization layer is required for any multi-turn conversation with thinking enabled.

**Evidence:** Open-source AI agent project — March 2026

## The underlying pattern
When routing provider-specific model variants through a generic OpenAI-compatible abstraction layer, provider-specific fields that exist outside the OpenAI schema will be silently dropped. For Gemini thinking models, the dropped field (`thought_signature`) is a required round-trip token that causes hard API rejections on subsequent turns. The fix must live at the serialization boundary — inspect and preserve the field before every outgoing request.
