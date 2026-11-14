# Exercise 2: Configure Claude Code for a Team Development Workflow

## Objective

Practice configuring CLAUDE.md hierarchies, custom slash commands, path-specific rules, and MCP server integration for a multi-developer project.

## Domains reinforced

[3](../../domains/03-claude-code-configuration-workflows/README.md), [2](../../domains/02-tool-design-mcp-integration/README.md)

## Steps

1. Create a project-level CLAUDE.md with universal coding standards and testing conventions. Verify that instructions at the project level are consistently applied across all team members.
2. Create `.claude/rules/` files with YAML frontmatter glob patterns for different code areas (e.g., `paths: ["src/api/**/*"]` for API conventions, `paths: ["**/*.test.*"]` for testing conventions). Test that rules load only when editing matching files.
3. Create a project-scoped skill in `.claude/skills/` with `context: fork` and `allowed-tools` restrictions. Verify the skill runs in isolation without polluting the main conversation context.
4. Configure an MCP server in `.mcp.json` with environment variable expansion for credentials. Add a personal experimental MCP server in `~/.claude.json` and verify both are available simultaneously.
5. Test plan mode versus direct execution on tasks of varying complexity: a single-file bug fix, a multi-file library migration, and a new feature with multiple valid implementation approaches. Observe when plan mode provides value.

## Worked solution

This is configuration, not application code. Use any small TypeScript repo with `src/api/`, `src/components/` and some `*.test.ts` files. Background: [3.1](../../domains/03-claude-code-configuration-workflows/3.1-claude-md-hierarchy.md) to [3.5](../../domains/03-claude-code-configuration-workflows/3.5-iterative-refinement.md), [2.4](../../domains/02-tool-design-mcp-integration/2.4-mcp-server-integration.md). A fuller layout is in the [Claude Code scenario](../../scenarios/02-code-generation-with-claude-code/README.md).

### Step 1: project `CLAUDE.md`

Commit this at the repo root so every clone gets it. Personal preferences go in `~/.claude/CLAUDE.md` (never shared) or `CLAUDE.local.md` (gitignored).

```markdown
# Project standards

## Commands
- Install: `pnpm install`
- Test: `pnpm test` (run before every commit)
- Lint: `pnpm lint`

## Conventions
- TypeScript strict mode; no `any` without a comment explaining why
- 2-space indentation
- Never commit secrets; read them from the environment

## Testing
- Every bug fix gets a regression test
- Prefer real objects over mocks; mock only network and clock
```

Verify:

```text
/memory           lists every CLAUDE.md in play and where it came from
git clone <repo> /tmp/fresh && cd /tmp/fresh && claude    # a fresh clone has none of your personal files, so only committed config can appear
/memory           the project CLAUDE.md is present in the fresh clone
```

Checklist:

- [ ] The standards are in the committed project file, not in `~/.claude/CLAUDE.md` (a teammate would never see those).
- [ ] A fresh clone shows the same instructions with no extra setup.

### Step 2: path-specific rules

```markdown
<!-- .claude/rules/api-conventions.md -->
---
paths:
  - "src/api/**/*"
---
# API handlers
- async/await with the shared `handleError` wrapper
- Validate input with zod before touching the database
- Return `{ error: { code, message } }` on failure
```

```markdown
<!-- .claude/rules/testing.md -->
---
paths:
  - "**/*.test.*"
---
# Test conventions
- Colocate tests next to the code under test
- One behaviour per test, Arrange / Act / Assert
- No snapshot tests
```

Test that they load only when relevant:

```text
> Read src/components/Button.tsx and tell me what conventions apply.
  (neither rule loaded: no matching file)
> Add a validation test to src/api/handlers/invoices.test.ts
  (both rules match: *.test.* and src/api/**)
> /memory
  (shows which rules are loaded)
```

Checklist:

- [ ] Editing `Button.tsx` does not load the API rules.
- [ ] Editing `invoices.test.ts` loads both rules.
- [ ] `claude --debug` shows no frontmatter parse errors. A rule whose frontmatter fails to parse loads on every task, which defeats the point.

Use rules (not directory-level CLAUDE.md) when a convention spans files in many directories, as test files do.

### Step 3: a forked skill with restricted tools

```yaml
# .claude/skills/codebase-analysis/SKILL.md
---
name: codebase-analysis
description: Analyse how a module is organised and summarise its dependencies and risks. Use when asked to understand an area of the codebase.
context: fork
agent: Explore
argument-hint: [module-path]
allowed-tools: Read Grep Glob
---

Analyse $ARGUMENTS:
1. Find entry points with Grep
2. Read the files they import
3. Return a summary: purpose, public API, dependencies, risks, with file references
```

Verify isolation:

```text
> /context            note the token count
> /codebase-analysis src/api
> /context            only the skill's summary was added, not every file it read   # the exploration cost stays in the fork
```

Checklist:

- [ ] `context: fork` runs the skill in a separate context; only its result returns to the main conversation.
- [ ] `allowed-tools` limits it to read-only tools. Ask it to edit a file and confirm it cannot.
- [ ] Contrast with `.claude/commands/`: a command (or a skill without `context: fork`) runs in the main context.

### Step 4: MCP servers, shared and personal

Shared, committed. The credential comes from each developer's environment, never from the file:

```json
{
  "mcpServers": {
    "github": {
      "type": "http",
      "url": "https://api.githubcopilot.com/mcp/",
      "headers": { "Authorization": "Bearer ${GITHUB_TOKEN}" }
    }
  }
}
```

Personal and experimental, stored in your user config (`~/.claude.json`) and never committed:

```text
claude mcp add --transport http hubspot --scope user https://mcp.hubspot.com/anthropic   # user scope: lives in ~/.claude.json, never in the repo
claude mcp list
```

```text
> /mcp          both servers appear in the same session
```

Checklist:

- [ ] `GITHUB_TOKEN` is set in your shell; the repo contains only `${GITHUB_TOKEN}`.
- [ ] Project (`.mcp.json`) and user-scope servers are available at the same time.
- [ ] Unset the variable and confirm the shared server fails to connect, rather than a secret being found in the repo.

### Step 5: plan mode versus direct execution

Start a session in plan mode with `claude --permission-mode plan` (or toggle with Shift+Tab), and use direct mode for the others.

| Task | Mode | Why |
| - | - | - |
| Fix a `NaN` in one function with a stack trace | Direct | Clear scope, one file, obvious fix; planning is overhead |
| Migrate a library across many files (e.g. `moment` to `date-fns`) | Plan, then direct | Many files, so explore first and agree the approach before editing |
| Add a feature with several valid designs (e.g. caching) | Plan, or interview first | Trade-offs matter; `> Interview me about invalidation, failure modes and TTLs before writing code` |

```text
claude --permission-mode plan
> Migrate all uses of moment to date-fns. Identify every call site, group them by pattern, and propose an order.
  (review the plan, then approve; Claude switches to editing)
```

Checklist:

- [ ] For the bug fix, a plan added nothing.
- [ ] For the migration, the plan exposed call-site patterns you had not listed.
- [ ] For the ambiguous feature, the interview surfaced a requirement (e.g. failure behaviour) you had not stated.

## Commentary: what each step teaches

Names and flags are illustrative; the exam tests the reasoning, not the exact parameter.

### Step 1: project `CLAUDE.md`

- **Concept:** Configuration has a scope that follows where it is stored. Team-wide standards belong in a file that version control distributes; personal preferences belong in files that are never shared.
- **What to observe:** A fresh clone, with none of the author's personal files, still shows the project instructions in `/memory`. If a teammate does not see a rule, it lives in user-level or gitignored configuration rather than the committed file. Inconsistent behaviour between teammates is a scoping problem before it is a wording problem.
- **Why it matters for the exam:** Builds "configuration the team must share lives in the repository" (Question 4 in the [Claude Code scenario](../../scenarios/02-code-generation-with-claude-code/README.md)).

### Step 2: path-specific rules

- **Concept:** Context costs tokens and dilutes attention, so conventions load only when the work touches matching files. A glob pattern scopes by file shape, which suits conventions spread across many directories.
- **What to observe:** `Button.tsx` loads neither rule; `invoices.test.ts` loads both, because it matches `*.test.*` and `src/api/**`. A rule that loads everywhere signals broken frontmatter (it then behaves as unconditional); a rule that never loads signals a pattern that does not match the path form used.
- **Why it matters for the exam:** Builds "scope config to who needs it" and "prefer deterministic scoping over model inference". A directory-level `CLAUDE.md` cannot cover tests scattered across the tree, and headers in a root file depend on the model inferring relevance (Question 6).

### Step 3: forked skill with restricted tools

- **Concept:** An exploration task reads many files that the main conversation does not need afterwards. Running it in a forked context keeps only the summary, and restricting tools applies least privilege to a task that should only read.
- **What to observe:** `/context` grows by the size of the summary, not by every file read. An edit attempt fails because the tool is not allowed. If the main context grows heavily, the skill is running inline (no fork). If the skill edits files, `allowed-tools` is missing or too broad.
- **Why it matters for the exam:** Builds "isolate verbose work, return only the result" and "restrict the capability to the task". The contrast with a command or an unforked skill, which share the main context, is a common question.

### Step 4: shared and personal MCP servers

- **Concept:** Server definitions are shared through the repository, secrets are not. The definition references an environment variable that each developer supplies, and experimental servers stay in personal scope.
- **What to observe:** `/mcp` shows both servers in one session. With the variable unset, the shared server fails to connect and no credential exists anywhere in the repository. A secret found in `.mcp.json` means the expansion pattern was not used. A personal server visible to teammates means it was added at project scope.
- **Why it matters for the exam:** Builds "scope config to who needs it" and "secrets stay outside version control". Tools from all configured servers are available together, so the design question is scope and credentials, not choosing one server.

### Step 5: plan mode versus direct execution

- **Concept:** Planning is exploration before commitment. It pays off when scope is large, several designs are valid, or a wrong first step is expensive to undo; it is overhead for a clear, single-file change.
- **What to observe:** The bug fix needs no plan. The migration plan lists call-site patterns that were not in the prompt. The feature interview surfaces unstated requirements such as failure behaviour. If planning added nothing to the migration, the task description probably already contained the exploration result.
- **Why it matters for the exam:** Builds "plan first when architectural complexity is stated, go direct when the scope is clear". Deferring the thinking until edits exist (Question 5 distractors) is the trap.

### Common mistakes

- Putting team conventions in `~/.claude/CLAUDE.md`: they work for the author and are invisible to everyone else.
- Writing glob frontmatter that fails to parse: the rule silently loads on every task, wasting context and applying API conventions to unrelated files.
- Using a `CLAUDE.md` per directory for test conventions: tests spread across the tree are not covered, and the convention is duplicated.
- Running a read-heavy analysis skill inline with all tools: the main context fills with file contents and the skill can modify the repository.
- Hard-coding a token in `.mcp.json`: the secret is committed and shared with everyone who clones.
- Using plan mode for every task, or direct mode for a cross-cutting migration: wasted overhead in the first case, a half-migrated codebase with an inconsistent approach in the second.

### Extension ideas

- Add a monorepo with two packages that need different conventions, and decide which parts belong in directory-level `CLAUDE.md` files and which in glob rules. Tests the rule that the scoping unit should match how the convention is distributed.
- Add a second rule that conflicts with the first on `*.test.*` files inside `src/api/`, and work out how precedence and explicit wording resolve it. Tests reasoning about overlapping scope instead of assuming one rule per file.
- Run the forked analysis skill in CI non-interactively and decide what output format a pipeline could consume. Tests carrying the same isolation and least-privilege ideas into an automated setting.
