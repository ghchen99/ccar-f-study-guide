# Scenario 2: Code Generation with Claude Code

## Description

You are using Claude Code to accelerate software development. Your team uses it for code generation, refactoring, debugging, and documentation. You need to integrate it into your development workflow with custom slash commands, CLAUDE.md configurations, and understand when to use plan mode vs direct execution.

## Primary domains

- [Domain 3: Claude Code Configuration & Workflows](../../domains/03-claude-code-configuration-workflows/README.md)
- [Domain 5: Context Management & Reliability](../../domains/05-context-management-reliability/README.md)

## Sample questions

### Question 4

You want to create a custom `/review` slash command that runs your team's standard code review checklist. This command should be available to every developer when they clone or pull the repository. Where should you create this command file?

- A. In the `.claude/commands/` directory in the project repository
- B. In `~/.claude/commands/` in each developer's home directory
- C. In the `CLAUDE.md` file at the project root
- D. In a `.claude/config.json` file with a `commands` array

**Correct: A.** Project-scoped custom slash commands live in `.claude/commands/` within the repository; they are version-controlled and automatically available to all developers. B is for personal commands not shared via version control. C is for project instructions and context, not command definitions. D describes a mechanism that doesn't exist in Claude Code.

Related: [3.2](../../domains/03-claude-code-configuration-workflows/3.2-slash-commands-skills.md)

### Question 5

You've been assigned to restructure the team's monolithic application into microservices. This will involve changes across dozens of files and requires decisions about service boundaries and module dependencies. Which approach should you take?

- A. Enter plan mode to explore the codebase, understand dependencies, and design an implementation approach before making changes.
- B. Start with direct execution and make changes incrementally, letting the implementation reveal the natural service boundaries.
- C. Use direct execution with comprehensive upfront instructions detailing exactly how each service should be structured.
- D. Begin in direct execution mode and only switch to plan mode if you encounter unexpected complexity during implementation.

**Correct: A.** Plan mode is designed for complex tasks involving large-scale changes, multiple valid approaches, and architectural decisions. It enables safe exploration and design before committing to changes. B risks costly rework when dependencies are discovered late. C assumes you already know the right structure without exploring the code. D ignores that the complexity is already stated in the requirements.

Related: [3.4](../../domains/03-claude-code-configuration-workflows/3.4-plan-mode-vs-direct.md)

### Question 6

Your codebase has distinct areas with different coding conventions: React components use functional style with hooks, API handlers use async/await with specific error handling, and database models follow a repository pattern. Test files are spread throughout the codebase alongside the code they test (e.g., `Button.test.tsx` next to `Button.tsx`), and you want all tests to follow the same conventions regardless of location. What's the most maintainable way to ensure Claude automatically applies the correct conventions when generating code?

- A. Create rule files in `.claude/rules/` with YAML frontmatter specifying glob patterns to conditionally apply conventions based on file paths
- B. Consolidate all conventions in the root `CLAUDE.md` file under headers for each area, relying on Claude to infer which section applies
- C. Create skills in `.claude/skills/` for each code type that include the relevant conventions in their `SKILL.md` files
- D. Place a separate `CLAUDE.md` file in each subdirectory containing that area's specific conventions

**Correct: A.** `.claude/rules/` with glob patterns (e.g., `**/*.test.tsx`) applies conventions based on file paths regardless of directory location, which is essential for test files spread throughout the codebase. B relies on inference, making it unreliable. C requires manual skill invocation or Claude choosing to load them, contradicting the need for automatic application. D can't easily handle files spread across many directories since CLAUDE.md files are directory-bound.

Related: [3.3](../../domains/03-claude-code-configuration-workflows/3.3-path-specific-rules.md), [3.1](../../domains/03-claude-code-configuration-workflows/3.1-claude-md-hierarchy.md)

## Code snippets

_To be added._
