# Domain 4: Prompt Engineering & Structured Output

**Weight:** 20%

Start with explicit, measurable criteria. Add examples and schemas only where they improve a demonstrated failure mode, then design validation and review loops.

## Revision sequence

1. Make acceptance criteria explicit and testable.
2. Use few-shot examples for genuine ambiguity rather than general instruction.
3. Choose schema or tool constraints appropriate to the downstream system.
4. Define validation, retries, and feedback that preserve failures for the model.
5. Process large workloads and reviews in controlled batches or passes.

## Task statements

- [4.1 Explicit criteria to improve precision](4.1-explicit-criteria.md)
- [4.2 Few-shot prompting](4.2-few-shot-prompting.md)
- [4.3 Structured output via tool use and JSON schemas](4.3-structured-output-tool-use.md)
- [4.4 Validation, retry, and feedback loops](4.4-validation-retry-loops.md)
- [4.5 Batch processing strategies](4.5-batch-processing.md)
- [4.6 Multi-instance and multi-pass review](4.6-multi-pass-review.md)

## Scenarios drawing on this domain

- [Scenario 5: Claude Code for Continuous Integration](../../scenarios/05-claude-code-for-ci/README.md)
- [Scenario 6: Structured Data Extraction](../../scenarios/06-structured-data-extraction/README.md)
