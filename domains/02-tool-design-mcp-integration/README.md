# Domain 2: Tool Design & MCP Integration

**Weight:** 18%

Start by making each tool unambiguous. Then define structured failures, tool-selection policies, MCP boundaries, and the role of built-in tools.

## Revision sequence

1. Design tool names, descriptions, schemas, and boundaries.
2. Define a consistent error contract for callers and agents.
3. Explain tool distribution and `tool_choice` trade-offs.
4. Separate MCP resource/server integration from application logic.
5. Decide when built-in tools are sufficient and when custom tools are needed.

## Task statements

- [2.1 Tool interface design](2.1-tool-interface-design.md)
- [2.2 Structured error responses for MCP tools](2.2-structured-error-responses.md)
- [2.3 Tool distribution and tool_choice](2.3-tool-distribution-tool-choice.md)
- [2.4 MCP server integration](2.4-mcp-server-integration.md)
- [2.5 Built-in tools](2.5-built-in-tools.md)

## Scenarios drawing on this domain

- [Scenario 1: Customer Support Resolution Agent](../../scenarios/01-customer-support-resolution-agent/README.md)
- [Scenario 3: Multi-Agent Research System](../../scenarios/03-multi-agent-research-system/README.md)
- [Scenario 4: Developer Productivity with Claude](../../scenarios/04-developer-productivity/README.md)
