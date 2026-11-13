# Scenario 5: Claude Code for Continuous Integration

## Description

You are integrating Claude Code into your Continuous Integration/Continuous Deployment (CI/CD) pipeline. The system runs automated code reviews, generates test cases, and provides feedback on pull requests. You need to design prompts that provide actionable feedback and minimize false positives.

## Primary domains

- [Domain 3: Claude Code Configuration & Workflows](../../domains/03-claude-code-configuration-workflows/README.md)
- [Domain 4: Prompt Engineering & Structured Output](../../domains/04-prompt-engineering-structured-output/README.md)

## Sample questions

### Question 10

Your pipeline script runs `claude "Analyze this pull request for security issues"` but the job hangs indefinitely. Logs indicate Claude Code is waiting for interactive input. What's the correct approach to run Claude Code in an automated pipeline?

- A. Add the `-p` flag: `claude -p "Analyze this pull request for security issues"`
- B. Set the environment variable `CLAUDE_HEADLESS=true` before running the command
- C. Redirect stdin from `/dev/null`: `claude "Analyze this pull request for security issues" < /dev/null`
- D. Add the `--batch` flag: `claude --batch "Analyze this pull request for security issues"`

**Correct: A.** `-p` (or `--print`) is the documented way to run Claude Code non-interactively: it processes the prompt, outputs to stdout, and exits. The other options reference non-existent features (`CLAUDE_HEADLESS`, `--batch`) or Unix workarounds that don't address Claude Code's command syntax.

Related: [3.6](../../domains/03-claude-code-configuration-workflows/3.6-ci-cd-integration.md)

### Question 11

Your team wants to reduce API costs for automated analysis. Currently, real-time Claude calls power two workflows: (1) a blocking pre-merge check that must complete before developers can merge, and (2) a technical debt report generated overnight for review the next morning. Your manager proposes switching both to the Message Batches API for its 50% cost savings. How should you evaluate this proposal?

- A. Use batch processing for the technical debt reports only; keep real-time calls for pre-merge checks.
- B. Switch both workflows to batch processing with status polling to check for completion.
- C. Keep real-time calls for both workflows to avoid batch result ordering issues.
- D. Switch both to batch processing with a timeout fallback to real-time if batches take too long.

**Correct: A.** The Batches API offers 50% savings but up to 24-hour processing with no guaranteed latency SLA, so it is unsuitable for blocking pre-merge checks but ideal for overnight jobs. B is wrong because relying on "often faster" completion isn't acceptable for blocking workflows. C reflects a misconception; results can be correlated with `custom_id`. D adds unnecessary complexity.

Related: [4.5](../../domains/04-prompt-engineering-structured-output/4.5-batch-processing.md)

### Question 12

A pull request modifies 14 files across the stock tracking module. Your single-pass review analyzing all files together produces inconsistent results: detailed feedback for some files but superficial comments for others, obvious bugs missed, and contradictory feedback (flagging a pattern as problematic in one file while approving identical code elsewhere in the same PR). How should you restructure the review?

- A. Split into focused passes: analyze each file individually for local issues, then run a separate integration-focused pass examining cross-file data flow.
- B. Require developers to split large PRs into smaller submissions of 3-4 files before the automated review runs.
- C. Switch to a higher-tier model with a larger context window to give all 14 files adequate attention in one pass.
- D. Run three independent review passes on the full PR and only flag issues that appear in at least two of the three runs.

**Correct: A.** Focused passes address the root cause: attention dilution. Per-file analysis ensures consistent depth; a separate integration pass catches cross-file issues. B shifts burden to developers without improving the system. C misunderstands that larger context windows don't solve attention quality. D would suppress real bugs by requiring consensus on issues that may only be caught intermittently.

Related: [4.6](../../domains/04-prompt-engineering-structured-output/4.6-multi-pass-review.md), [1.6](../../domains/01-agentic-architecture-orchestration/1.6-task-decomposition.md)

## Code snippets

_To be added._
