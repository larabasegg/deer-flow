---
name: large-tool-registry-deferred-schema
description: "Prevents context window inflation and tool selection degradation when an agent connects to many tools. TRIGGER when: registering more than 10-15 tools in any agent tool system (MCP servers, OpenAI function calling, Anthropic tool_use, custom registries), noticing degraded tool selection accuracy as the tool set grows, context costs increasing disproportionately with each new tool added, adding dynamic tool discovery to a large-tool-set agent."
---

## Key Decisions

1. **When the tool set grows beyond roughly 10-15 tools, withhold full tool schemas from the LLM context — expose only tool names and short descriptions, and load the full schema for a specific tool only when the model selects it by name.** Each tool schema can run hundreds of tokens; a large tool registry included in full inflates the context window proportionally with tool count, increasing cost and diluting the model's attention across too many option descriptions simultaneously.

2. **Apply eager schema loading (full schemas in context) for small tool sets; switch to name-only discovery with on-demand schema loading when the tool set grows or when tool selection accuracy degrades.** The crossover is not purely about token count — it is also about whether the fraction of context consumed by tool descriptions degrades the quality of reasoning and tool selection. Eager loading remains correct and simpler below the threshold; switching to deferred loading above it requires a mechanism for the model to request a schema before using a tool.

## Anti-patterns

- **What**: Including all tool schemas in every LLM context window regardless of how many tools are registered.
  **Why**: Schema inclusion scales linearly with tool count; with a large registry, tool schemas may consume most of the available context window.
  **Symptom**: Tool selection accuracy drops as more tools are added; the model selects incorrect tools or ignores available tools positioned late in a long schema list; cost grows without a corresponding increase in capability.

- **What**: Switching to on-demand schema loading without providing the model a list of available tool names.
  **Why**: If the model receives no tool listing, it cannot know which tools to request a schema for and will not use tools at all.
  **Symptom**: Agent stops using tools entirely after the schema loading mode changes; responses are pure text where tool calls would have been appropriate.

## Structural Template

```
EAGER_THRESHOLD = 15  # adjust based on observed context quality

class ToolRegistry:

    def get_context_payload(self, mode="auto"):
        tools = self.load_all_tools()

        if mode == "auto":
            mode = "eager" if len(tools) <= EAGER_THRESHOLD else "deferred"

        if mode == "eager":
            # Full schemas — correct for small tool sets
            return [tool.full_schema() for tool in tools]
        else:
            # Names + one-line descriptions only
            # Model must call load_schema_on_demand() before invoking a tool
            return [tool.name_and_description() for tool in tools]

    def load_schema_on_demand(self, tool_name):
        # Invoked when model emits a schema-lookup / tool_search call
        return self.get_tool(tool_name).full_schema()
```

*Evidence from: MCP tool servers in a multi-agent research system (open-source AI agent project, March 2026)*
