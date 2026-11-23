# Exam overview

**Claude Certified Architect – Foundations**  
Exam code: **CCAR-F** · Blueprint version: **1.0** · Effective: **July 2026**

[Begin revision](domains/01-agentic-architecture-orchestration/README.md)

## What the exam measures

The exam validates architectural judgement across Claude Code, the Claude Agent SDK, the Claude API, and Model Context Protocol (MCP). Questions are scenario-based and assess whether you can choose the right trade-off for a realistic production system.

You should be prepared to:

- Build and coordinate agentic systems
- Integrate tools and MCP servers
- Configure Claude Code for team workflows
- Produce reliable structured output
- Manage context, uncertainty, escalation, and human review
- Explain why a pattern is appropriate, not just how to implement it

## Who it is designed for

Typically, a solution architect or experienced Claude practitioner with hands-on exposure to agent SDKs, Claude Code configuration, MCP integration, prompt engineering, CI/CD, and production reliability patterns.

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

Use three passes: learn the task statements, apply them to scenarios, then retrieve the decisions from memory.

### First pass: learn the blueprint

- Work through each domain's task statements in the order of its index.
- For every statement, distinguish the **model-driven** approach from a **programmatic** safeguard.
- Record trade-offs, not just preferred answers.
- Build or configure one small example for each pattern before moving on.

### Second pass: practise decisions

- Attempt every scenario's questions with the answers hidden.
- Explain why each distractor fails and link the answer to a domain task statement.
- Complete the four exercises, starting from the [exercise index](exercises/README.md).
- Practise the following scenarios:
  - Agent loop and tool-calling reliability
  - Claude Code team configuration
  - Structured extraction and validation
  - Multi-agent research and context handoff

### Final pass: retrieval and review

- Answer the "check yourself" questions without viewing the surrounding notes.
- For missed questions, return to the relevant task statement and produce a one-sentence rule.
- Review the exam traps in the domain task-statement pages only after practising the normal answer first.
- Revisit every domain with weak check-yourself results before scheduling the exam.

## Revision checklist

- [ ] Can explain `stop_reason` and the agentic tool-use loop
- [ ] Can choose between prompts, routing, and programmatic workflow enforcement
- [ ] Can design overlapping MCP tools with clear boundaries
- [ ] Can structure tool errors and handle retryable failures
- [ ] Can explain CLAUDE.md scope, path-specific rules, and skills
- [ ] Can choose plan mode versus direct execution
- [ ] Can implement schema-constrained output and validation retries
- [ ] Can preserve facts, provenance, and uncertainty across context
- [ ] Can define escalation criteria and human-review thresholds
- [ ] Have completed all four hands-on exercises
- [ ] Have attempted all six scenarios and explained every distractor
