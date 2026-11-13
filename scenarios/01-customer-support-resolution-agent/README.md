# Scenario 1: Customer Support Resolution Agent

## Description

You are building a customer support resolution agent using the Claude Agent SDK. The agent handles high-ambiguity requests like returns, billing disputes, and account issues. It has access to your backend systems through custom Model Context Protocol (MCP) tools (`get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`). Your target is 80%+ first-contact resolution while knowing when to escalate.

## Primary domains

- [Domain 1: Agentic Architecture & Orchestration](../../domains/01-agentic-architecture-orchestration/README.md)
- [Domain 2: Tool Design & MCP Integration](../../domains/02-tool-design-mcp-integration/README.md)
- [Domain 5: Context Management & Reliability](../../domains/05-context-management-reliability/README.md)

## Sample questions

### Question 1

Production data shows that in 12% of cases, your agent skips `get_customer` entirely and calls `lookup_order` using only the customer's stated name, occasionally leading to misidentified accounts and incorrect refunds. What change would most effectively address this reliability issue?

- A. Add a programmatic prerequisite that blocks `lookup_order` and `process_refund` calls until `get_customer` has returned a verified customer ID.
- B. Enhance the system prompt to state that customer verification via `get_customer` is mandatory before any order operations.
- C. Add few-shot examples showing the agent always calling `get_customer` first, even when customers volunteer order details.
- D. Implement a routing classifier that analyzes each request and enables only the subset of tools appropriate for that request type.

**Correct: A.** When a specific tool sequence is required for critical business logic (like verifying customer identity before processing refunds), programmatic enforcement provides deterministic guarantees that prompt-based approaches cannot. B and C rely on probabilistic LLM compliance, which is insufficient when errors have financial consequences. D addresses tool availability rather than tool ordering, which is not the actual problem.

Related: [1.4](../../domains/01-agentic-architecture-orchestration/1.4-workflow-enforcement-handoff.md), [1.5](../../domains/01-agentic-architecture-orchestration/1.5-hooks-interception-normalization.md)

### Question 2

Production logs show the agent frequently calls `get_customer` when users ask about orders (e.g., "check my order #12345"), instead of calling `lookup_order`. Both tools have minimal descriptions ("Retrieves customer information" / "Retrieves order details") and accept similar identifier formats. What's the most effective first step to improve tool selection reliability?

- A. Add few-shot examples to the system prompt demonstrating correct tool selection patterns, with 5-8 examples showing order-related queries routing to `lookup_order`.
- B. Expand each tool's description to include input formats it handles, example queries, edge cases, and boundaries explaining when to use it versus similar tools.
- C. Implement a routing layer that parses user input before each turn and pre-selects the appropriate tool based on detected keywords and identifier patterns.
- D. Consolidate both tools into a single `lookup_entity` tool that accepts any identifier and internally determines which backend to query.

**Correct: B.** Tool descriptions are the primary mechanism LLMs use for tool selection. When descriptions are minimal, models lack the context to differentiate between similar tools. B addresses this root cause with a low-effort, high-leverage fix. Few-shot examples (A) add token overhead without fixing the underlying issue. A routing layer (C) is over-engineered and bypasses the LLM's natural language understanding. Consolidating tools (D) is a valid architectural choice but requires more effort than a "first step" warrants.

Related: [2.1](../../domains/02-tool-design-mcp-integration/2.1-tool-interface-design.md)

### Question 3

Your agent achieves 55% first-contact resolution, well below the 80% target. Logs show it escalates straightforward cases (standard damage replacements with photo evidence) while attempting to autonomously handle complex situations requiring policy exceptions. What's the most effective way to improve escalation calibration?

- A. Add explicit escalation criteria to your system prompt with few-shot examples demonstrating when to escalate versus resolve autonomously.
- B. Have the agent self-report a confidence score (1-10) before each response and automatically route requests to humans when confidence falls below a threshold.
- C. Deploy a separate classifier model trained on historical tickets to predict which requests need escalation before the main agent begins processing.
- D. Implement sentiment analysis to detect customer frustration levels and automatically escalate when negative sentiment exceeds a threshold.

**Correct: A.** Explicit escalation criteria with few-shot examples directly address the root cause: unclear decision boundaries. This is the proportionate first response before adding infrastructure. B fails because LLM self-reported confidence is poorly calibrated; the agent is already incorrectly confident on hard cases. C is over-engineered, requiring labeled data and ML infrastructure when prompt optimization hasn't been tried. D solves a different problem; sentiment doesn't correlate with case complexity.

Related: [5.2](../../domains/05-context-management-reliability/5.2-escalation-ambiguity.md), [4.2](../../domains/04-prompt-engineering-structured-output/4.2-few-shot-prompting.md)

## Code snippets

_To be added._
