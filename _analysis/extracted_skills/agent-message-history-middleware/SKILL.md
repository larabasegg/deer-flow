---
name: agent-message-history-middleware
description: "Maintains message history invariants required by LLM APIs when middleware injects or repairs messages mid-conversation. TRIGGER when: building middleware that injects messages into an ongoing conversation, handling agent interruption or cancellation that leaves tool calls without responses, injecting warnings or status messages mid-conversation across multiple LLM providers."
---

## Key Decisions

1. **When injecting status or warning messages mid-conversation, use a user-role message, not a system-role message.** Some LLM providers restrict system-role messages to the conversation start — injecting a system message after the first user turn causes API-level errors in the provider's message formatter. A user-role message is accepted universally across providers and carries the same semantic weight for the model.

2. **When a running agent is interrupted before all tool call responses arrive, inject placeholder responses for every unpaired tool call before the next model invocation.** LLM APIs enforce strict tool call / tool response pairing — a message history containing tool calls with no corresponding responses is rejected at the API level. The placeholder must be syntactically valid but can carry an interruption notice rather than real tool output.

## Anti-patterns

- **What**: Injecting system-role messages mid-conversation to deliver runtime warnings or loop-detection notices.
  **Why**: Some provider message formatters require system messages to appear only at the start of the conversation history.
  **Symptom**: Works in local testing with one LLM provider; fails with a formatter or API error on a different provider, with an error message pointing at message type rather than the logic that injected it.

- **What**: Resuming an interrupted agent without repairing unpaired tool call entries in message history.
  **Why**: Tool calls and tool responses in the message history must be paired; an unpaired tool call causes the LLM API to reject the entire history.
  **Symptom**: After a user interruption, every subsequent agent invocation fails with an API validation error about malformed tool call structure, requiring a full conversation reset to recover.

## Structural Template

```
class MessageHistoryMiddleware:

    def before_model_call(self, history):
        history = self._repair_dangling_tool_calls(history)
        return history

    def inject_notice(self, history, notice_text):
        # Always user-role — never system-role for mid-conversation injections
        history.append(UserMessage(content=notice_text))
        return history

    def _repair_dangling_tool_calls(self, history):
        pending_calls = find_tool_calls_without_responses(history)
        for call in pending_calls:
            placeholder = ToolMessage(
                tool_call_id=call.id,
                content="[interrupted — no response captured]"
            )
            history.append(placeholder)
        return history


function find_tool_calls_without_responses(history):
    calls     = {msg.tool_call_id: msg for msg in history if is_tool_call(msg)}
    responded = {msg.tool_call_id      for msg in history if is_tool_response(msg)}
    return [calls[id] for id in calls if id not in responded]
```
