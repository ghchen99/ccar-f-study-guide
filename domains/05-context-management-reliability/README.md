# Domain 5: Context Management & Reliability

**Weight:** 15%

Start with what must survive a long interaction. Then control escalation, error propagation, codebase context, human review, provenance, and uncertainty.

## Revision sequence

1. Preserve only the critical facts needed for the next decision.
2. Define escalation triggers and ambiguity handling.
3. Propagate structured errors and partial results across agents.
4. Explore large codebases with scoped evidence rather than dumping context.
5. Route uncertain outputs to calibrated human review with traceable sources.

## Task statements

- [5.1 Preserve critical information across long interactions](5.1-context-preservation.md)
- [5.2 Escalation and ambiguity resolution](5.2-escalation-ambiguity.md)
- [5.3 Error propagation across multi-agent systems](5.3-error-propagation.md)
- [5.4 Context in large codebase exploration](5.4-codebase-exploration-context.md)
- [5.5 Human review workflows and confidence calibration](5.5-human-review-confidence.md)
- [5.6 Provenance and uncertainty in multi-source synthesis](5.6-provenance-uncertainty.md)

## Scenarios drawing on this domain

- [Scenario 1: Customer Support Resolution Agent](../../scenarios/01-customer-support-resolution-agent/README.md)
- [Scenario 2: Code Generation with Claude Code](../../scenarios/02-code-generation-with-claude-code/README.md)
- [Scenario 3: Multi-Agent Research System](../../scenarios/03-multi-agent-research-system/README.md)
- [Scenario 6: Structured Data Extraction](../../scenarios/06-structured-data-extraction/README.md)
