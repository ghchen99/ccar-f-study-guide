# Exam Overview

Claude Certified Architect – Foundations · Exam code: CCAR-F · Version 1.0 (effective July 2026).

Source: [GUIDE.md](GUIDE.md)

## About the certification

Validates that practitioners can make informed decisions about tradeoffs when implementing real-world solutions with Claude. Covers Claude Code, the Claude Agent SDK, the Claude API, and Model Context Protocol (MCP). Questions are scenario-based, drawn from real customer use cases, and test practical judgment about architecture, configuration, and tradeoffs in production.

## Intended audience

A solution architect who designs and implements production applications with Claude, typically with 6+ months of hands-on experience in:

- Agentic applications with the Claude Agent SDK (multi-agent orchestration, subagent delegation, tool integration, lifecycle hooks)
- Claude Code for team workflows (CLAUDE.md, Agent Skills, MCP server integrations, plan mode)
- MCP tool and resource interface design for backend integration
- Prompt engineering for reliable structured output (JSON schemas, few-shot, extraction patterns)
- Context window management across long documents, multi-turn conversations, and multi-agent handoffs
- CI/CD integration for code review, test generation, and PR feedback
- Escalation and reliability decisions (error handling, human-in-the-loop, self-evaluation)

## Blueprint

| # | Domain | Weight |
|---|--------|--------|
| 1 | [Agentic Architecture & Orchestration](domains/01-agentic-architecture-orchestration/README.md) | 27% |
| 2 | [Tool Design & MCP Integration](domains/02-tool-design-mcp-integration/README.md) | 18% |
| 3 | [Claude Code Configuration & Workflows](domains/03-claude-code-configuration-workflows/README.md) | 20% |
| 4 | [Prompt Engineering & Structured Output](domains/04-prompt-engineering-structured-output/README.md) | 20% |
| 5 | [Context Management & Reliability](domains/05-context-management-reliability/README.md) | 15% |

## Exam format

4 scenarios are presented, picked at random from the 6 below.

| Scenario | Primary domains |
|----------|-----------------|
| [1. Customer Support Resolution Agent](scenarios/01-customer-support-resolution-agent/README.md) | 1, 2, 5 |
| [2. Code Generation with Claude Code](scenarios/02-code-generation-with-claude-code/README.md) | 3, 5 |
| [3. Multi-Agent Research System](scenarios/03-multi-agent-research-system/README.md) | 1, 2, 5 |
| [4. Developer Productivity with Claude](scenarios/04-developer-productivity/README.md) | 2, 3, 1 |
| [5. Claude Code for Continuous Integration](scenarios/05-claude-code-for-ci/README.md) | 3, 4 |
| [6. Structured Data Extraction](scenarios/06-structured-data-extraction/README.md) | 4, 5 |

## How to prepare

- Build an agent with the Claude Agent SDK: complete agentic loop with tool calling, error handling, and session management. Practice spawning subagents and passing context between them.
- Configure Claude Code for a real project: CLAUDE.md hierarchy, path-specific rules in `.claude/rules/`, custom skills with frontmatter (`context: fork`, `allowed-tools`), and at least one MCP server.
- Design and test MCP tools: descriptions that differentiate similar tools, structured error responses with categories and retryable flags, tool selection reliability on ambiguous requests.
- Build a structured data extraction pipeline: `tool_use` with JSON schemas, validation-retry loops, optional/nullable fields, Message Batches API.
- Practice prompt engineering: few-shot examples for ambiguous scenarios, explicit review criteria to reduce false positives, multi-pass review for large code reviews.
- Study context management: extracting structured facts from verbose tool outputs, scratchpad files for long sessions, subagent delegation to manage context limits.
- Review escalation and human-in-the-loop patterns: when to escalate (policy gaps, customer requests, inability to progress) vs resolve autonomously; confidence-based routing for human review.

Hands-on practice: see [exercises.md](exercises.md).
