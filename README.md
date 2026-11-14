# claude-cert-revision

Revision notes for the Claude Certified Architect – Foundations exam (CCAR-F), reorganised from the original [GUIDE.md](GUIDE.md).

## Structure

- [exam-overview.md](exam-overview.md): about, audience, blueprint, scenario list, how to prepare
- [domains/](domains/): theory and SDK knowledge, one folder per domain, one file per task statement
  1. [Agentic Architecture & Orchestration](domains/01-agentic-architecture-orchestration/README.md) (27%)
  2. [Tool Design & MCP Integration](domains/02-tool-design-mcp-integration/README.md) (18%)
  3. [Claude Code Configuration & Workflows](domains/03-claude-code-configuration-workflows/README.md) (20%)
  4. [Prompt Engineering & Structured Output](domains/04-prompt-engineering-structured-output/README.md) (20%)
  5. [Context Management & Reliability](domains/05-context-management-reliability/README.md) (15%)
- [scenarios/](scenarios/): the 6 exam scenarios, each with its sample questions
  1. [Customer Support Resolution Agent](scenarios/01-customer-support-resolution-agent/README.md)
  2. [Code Generation with Claude Code](scenarios/02-code-generation-with-claude-code/README.md)
  3. [Multi-Agent Research System](scenarios/03-multi-agent-research-system/README.md)
  4. [Developer Productivity with Claude](scenarios/04-developer-productivity/README.md)
  5. [Claude Code for Continuous Integration](scenarios/05-claude-code-for-ci/README.md)
  6. [Structured Data Extraction](scenarios/06-structured-data-extraction/README.md)
- [exercises/](exercises/README.md): the 4 hands-on preparation exercises, each with a worked solution
  1. [Multi-tool agent with escalation logic](exercises/01-multi-tool-agent-escalation/README.md)
  2. [Claude Code configuration for a team](exercises/02-claude-code-team-workflow/README.md)
  3. [Structured data extraction pipeline](exercises/03-structured-extraction-pipeline/README.md)
  4. [Multi-agent research pipeline](exercises/04-multi-agent-research-pipeline/README.md)

Each task statement in `domains/` has Python code snippets (Agent SDK, Messages API, Claude Code config files). Each scenario README has an end-to-end example that combines them.

The exam tests concepts and architecture, not SDK syntax. Every code file ends with a commentary section (design decision, walk-through, trade-offs, exam traps, check-yourself questions), so read the commentary before the code.
