# Contributing

Contributions that improve accuracy, clarity, and exam preparation are welcome.

## What to contribute

- Corrections to code examples or linked task statements
- Clearer explanations of trade-offs
- New or improved "check yourself" questions
- Small runnable examples that preserve the existing pedagogical structure
- Broken links, formatting fixes, and accessibility improvements

For current exam behaviour or SDK APIs, link an official Anthropic or Claude source.

## Content conventions

- Keep task-statement pages concise and organised around: an `At a glance` callout, `Example` (code), `How it works` (principle callout plus short numbered steps), `Trade-offs` (table or short bullets), an `Exam traps` warning callout, and `Check yourself` questions in collapsible `<details>` blocks. Use GitHub callouts (`> [!NOTE]`, `> [!IMPORTANT]`, `> [!WARNING]`, `> [!TIP]`) sparingly, and use `1.1-agentic-loops.md` as the reference.
- State SDK version assumptions when an API differs across releases.
- Treat model IDs and SDK names as illustrative unless a page explicitly records a required version.
- Prefer scenario-led explanations over isolated implementation details.
- Do not add vendor-specific claims without a source.
- Keep public examples free of credentials, customer data, and private endpoints.

## Pull requests

Before submitting:

1. Verify every relative Markdown link.
2. Check headings and domain/scenario cross-references.
3. Run or clearly mark code examples that require API credentials.
4. Keep changes focused; separate a content correction from a structural redesign.
5. Explain how the change improves revision or production judgement.
