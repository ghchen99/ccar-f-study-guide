# Scenario 3: Multi-Agent Research System

## Description

You are building a multi-agent research system using the Claude Agent SDK. A coordinator agent delegates to specialized subagents: one searches the web, one analyzes documents, one synthesizes findings, and one generates reports. The system researches topics and produces comprehensive, cited reports.

## Primary domains

- [Domain 1: Agentic Architecture & Orchestration](../../domains/01-agentic-architecture-orchestration/README.md)
- [Domain 2: Tool Design & MCP Integration](../../domains/02-tool-design-mcp-integration/README.md)
- [Domain 5: Context Management & Reliability](../../domains/05-context-management-reliability/README.md)

## Sample questions

### Question 7

After running the system on the topic "impact of AI on creative industries," you observe that each subagent completes successfully: the web search agent finds relevant articles, the document analysis agent summarizes papers correctly, and the synthesis agent produces coherent output. However, the final reports cover only visual arts, completely missing music, writing, and film production. When you examine the coordinator's logs, you see it decomposed the topic into three subtasks: "AI in digital art creation," "AI in graphic design," and "AI in photography." What is the most likely root cause?

- A. The synthesis agent lacks instructions for identifying coverage gaps in the findings it receives from other agents.
- B. The coordinator agent's task decomposition is too narrow, resulting in subagent assignments that don't cover all relevant domains of the topic.
- C. The web search agent's queries are not comprehensive enough and need to be expanded to cover more creative industry sectors.
- D. The document analysis agent is filtering out sources related to non-visual creative industries due to overly restrictive relevance criteria.

**Correct: B.** The coordinator's logs reveal the root cause: it decomposed "creative industries" into only visual arts subtasks. The subagents executed their assigned tasks correctly; the problem is what they were assigned. A, C, and D blame downstream agents that are working correctly within their scope.

Related: [1.2](../../domains/01-agentic-architecture-orchestration/1.2-coordinator-subagent-patterns.md)

### Question 8

The web search subagent times out while researching a complex topic. You need to design how this failure information flows back to the coordinator agent. Which error propagation approach best enables intelligent recovery?

- A. Return structured error context to the coordinator including the failure type, the attempted query, any partial results, and potential alternative approaches.
- B. Implement automatic retry logic with exponential backoff within the subagent, returning a generic "search unavailable" status only after all retries are exhausted.
- C. Catch the timeout within the subagent and return an empty result set marked as successful.
- D. Propagate the timeout exception directly to a top-level handler that terminates the entire research workflow.

**Correct: A.** Structured error context lets the coordinator decide whether to retry with a modified query, try an alternative approach, or proceed with partial results. B's generic status hides valuable context. C suppresses the error by marking failure as success, preventing recovery. D terminates the workflow unnecessarily.

Related: [5.3](../../domains/05-context-management-reliability/5.3-error-propagation.md), [2.2](../../domains/02-tool-design-mcp-integration/2.2-structured-error-responses.md)

### Question 9

During testing, you observe that the synthesis agent frequently needs to verify specific claims while combining findings. Currently, when verification is needed, the synthesis agent returns control to the coordinator, which invokes the web search agent, then re-invokes synthesis with results. This adds 2-3 round trips per task and increases latency by 40%. Your evaluation shows that 85% of these verifications are simple fact-checks (dates, names, statistics) while 15% require deeper investigation. What's the most effective approach to reduce overhead while maintaining system reliability?

- A. Give the synthesis agent a scoped `verify_fact` tool for simple lookups, while complex verifications continue delegating to the web search agent through the coordinator.
- B. Have the synthesis agent accumulate all verification needs and return them as a batch to the coordinator at the end of its pass, which then sends them all to the web search agent at once.
- C. Give the synthesis agent access to all web search tools so it can handle any verification need directly without round-trips through the coordinator.
- D. Have the web search agent proactively cache extra context around each source during initial research, anticipating what the synthesis agent might need to verify.

**Correct: A.** Applies least privilege: the synthesis agent gets only what it needs for the 85% common case while the existing coordination pattern handles complex cases. B creates blocking dependencies since synthesis steps may depend on earlier verified facts. C over-provisions the synthesis agent, violating separation of concerns. D relies on speculative caching that cannot reliably predict what needs verifying.

Related: [2.3](../../domains/02-tool-design-mcp-integration/2.3-tool-distribution-tool-choice.md)

## Code snippets

_To be added._
