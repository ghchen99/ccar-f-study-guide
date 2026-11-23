# Claude Certified Architect – Foundations

**CCAR-F revision course**

A public, scenario-led guide to the concepts, trade-offs, and implementation patterns tested by the exam.

> [!IMPORTANT]
> This course prioritises architectural judgement over SDK memorisation. Answer *why* a design works before asking *how* to write it.

## Choose your starting point

### Learning the syllabus

Start with the [exam overview](exam-overview.md), then work through the domains. The domain pages explain the task statements, implementation patterns, trade-offs, and common exam traps.

### Practising scenario decisions

Open the [scenario practice](scenarios/README.md). Attempt each question first, then follow the linked explanations and implementation patterns.

### Building hands-on experience

Use the [guided exercises](exercises/README.md) to construct agents and workflows from the ground up.

## A guided revision sequence

### 1 — Understand the foundations

1. [Agentic Architecture & Orchestration](domains/01-agentic-architecture-orchestration/README.md) — 27%
2. [Tool Design & MCP Integration](domains/02-tool-design-mcp-integration/README.md) — 18%

These domains establish the loop, orchestration patterns, tool contracts, MCP boundaries, and safeguards that recur throughout the exam.

### 2 — Learn the Claude Code workflow

3. [Claude Code Configuration & Workflows](domains/03-claude-code-configuration-workflows/README.md) — 20%

This domain connects repository configuration, path-specific behaviour, plan mode, iterative refinement, and CI/CD integration.

### 3 — Make outputs reliable

4. [Prompt Engineering & Structured Output](domains/04-prompt-engineering-structured-output/README.md) — 20%
5. [Context Management & Reliability](domains/05-context-management-reliability/README.md) — 15%

These domains cover schemas, validation, retries, context preservation, escalation, human review, provenance, and uncertainty.

### 4 — Apply the knowledge

- Work through the six [exam scenarios](scenarios/README.md).
- Complete the four [hands-on exercises](exercises/README.md).
- Revisit weak task statements from the domain pages.

## Repository guide

- [Exam overview](exam-overview.md) — what the exam measures and how to prepare
- [Learn: domains](domains/01-agentic-architecture-orchestration/README.md) — syllabus notes with code and review prompts
- [Practise: scenarios](scenarios/README.md) — realistic exam-style decisions
- [Build: exercises](exercises/README.md) — implementation from steps to worked solutions

## Using the code examples

Examples demonstrate architecture and SDK usage. Verify package versions and model IDs against the official Claude documentation before running them.

## Contributing

Corrections and improvements are welcome. Read [CONTRIBUTING.md](CONTRIBUTING.md) before opening a change.

## Licence

Licensed under the [MIT Licence](LICENSE).
