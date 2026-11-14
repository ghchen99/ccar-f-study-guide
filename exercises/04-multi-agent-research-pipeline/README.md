# Exercise 4: Design and Debug a Multi-Agent Research Pipeline

## Objective

Practice orchestrating subagents, managing context passing, implementing error propagation, and handling synthesis with provenance tracking.

## Domains reinforced

[1](../../domains/01-agentic-architecture-orchestration/README.md), [2](../../domains/02-tool-design-mcp-integration/README.md), [5](../../domains/05-context-management-reliability/README.md)

## Steps

1. Build a coordinator agent that delegates to at least two subagents (e.g., web search and document analysis). Ensure the coordinator's `allowedTools` includes `"Task"` and that each subagent receives its research findings directly in its prompt rather than relying on automatic context inheritance.
2. Implement parallel subagent execution by having the coordinator emit multiple Task tool calls in a single response. Measure the latency improvement compared to sequential execution.
3. Design structured output for subagents that separates content from metadata: each finding should include a claim, evidence excerpt, source URL/document name, and publication date. Verify that the synthesis subagent preserves source attribution when combining findings.
4. Implement error propagation: simulate a subagent timeout and verify the coordinator receives structured error context (failure type, attempted query, partial results). Test that the coordinator can proceed with partial results and annotate the final output with coverage gaps.
5. Test with conflicting source data (e.g., two credible sources with different statistics) and verify the synthesis output preserves both values with source attribution rather than arbitrarily selecting one, and structures the report to distinguish well-established from contested findings.

## Worked solution

The full system is in the [multi-agent research scenario](../../scenarios/03-multi-agent-research-system/README.md). This builds it up in the order of the steps, with a test for each. Background: [1.2](../../domains/01-agentic-architecture-orchestration/1.2-coordinator-subagent-patterns.md), [1.3](../../domains/01-agentic-architecture-orchestration/1.3-subagent-invocation-context-spawning.md), [2.2](../../domains/02-tool-design-mcp-integration/2.2-structured-error-responses.md), [5.3](../../domains/05-context-management-reliability/5.3-error-propagation.md), [5.6](../../domains/05-context-management-reliability/5.6-provenance-uncertainty.md).

The exam guide says `Task`. In current SDK versions the tool is named `Agent` and `Task` is kept as an alias, so use `"Agent"` in `allowed_tools` and check which name appears in your logs.

### Shared setup: stub tools with injectable failures and conflicts

```python
import asyncio
import json
import time
from typing import Any

from claude_agent_sdk import (
    AgentDefinition, ClaudeAgentOptions, HookMatcher, ResultMessage, create_sdk_mcp_server, query, tool,
)

FAIL_QUERIES: set[str] = set()      # step 4: add a query keyword to simulate a timeout
LATENCY = 3.0                        # seconds per stubbed search, so parallel vs sequential is measurable


def reply(payload: dict, is_error: bool = False) -> dict[str, Any]:
    out: dict[str, Any] = {"content": [{"type": "text", "text": json.dumps(payload)}]}
    if is_error:
        out["is_error"] = True
    return out


# two credible sources that disagree (step 5)
SOURCES = {
    "market size": [
        {"claim": "The AI-in-music market was $2.1B in 2024", "evidence_excerpt": "…reached $2.1 billion…",
         "source_url": "https://example.com/analyst-report", "document_name": None, "published": "2025-02-01"},
        {"claim": "The AI-in-music market was $3.4B in 2024", "evidence_excerpt": "…estimated at $3.4 billion…",
         "source_url": "https://example.org/industry-survey", "document_name": None, "published": "2025-05-15"},
    ],
    "adoption": [
        {"claim": "58% of studios use generative tools", "evidence_excerpt": "…58% of respondents…",
         "source_url": "https://example.com/studio-survey", "document_name": None, "published": "2025-03-10"},
    ],
}


@tool("web_search",
      "Search the web for ONE sub-topic. Returns findings with claim, evidence_excerpt, source_url, published. "
      "On failure returns an error object with failure_type, attempted_query, partial_results and alternatives.",
      {"query": str})
async def web_search(args):
    q = args["query"].lower()
    await asyncio.sleep(LATENCY)
    if any(k in q for k in FAIL_QUERIES):
        # Failure type, query and partial results let the coordinator choose retry, alternative or proceed.
        return reply({"status": "error", "failure_type": "timeout", "attempted_query": args["query"],
                      "partial_results": [], "alternatives": ["narrower query", "try doc-analyzer"]}, is_error=True)
    hits = [f for key, fs in SOURCES.items() if key in q for f in fs]
    return reply({"status": "ok", "results": hits})            # an empty list is a valid result, not an error


@tool("load_document", "Load a provided document by name and return its text with page markers.", {"name": str})
async def load_document(args):
    await asyncio.sleep(LATENCY)
    return reply({"document_name": args["name"], "pages": [{"page": 3, "text": "Studios report 61% adoption (n=400)."}]})


server = create_sdk_mcp_server(name="research", tools=[web_search, load_document])
```

### Step 3 first: the structured finding format

Every subagent returns the same shape, so the synthesizer never has to guess where a claim came from.

```python
FINDING_FORMAT = (
    'Return ONLY a JSON list. Each finding: {"claim", "evidence_excerpt", "source_url" | null, "document_name" | null, '
    '"page" | null, "published" (ISO date) | null, "methodology" | null}. '
    "Do not merge or average figures from different sources; if two sources differ, return both."
)

agents = {
    "web-searcher": AgentDefinition(
        description="Searches the web for one sub-topic and returns structured findings.",
        prompt="Research the sub-topic you are given with web_search. " + FINDING_FORMAT +
               " If the tool returns an error object, return it unchanged together with any partial results.",
        tools=["mcp__research__web_search"],                  # least privilege: each worker gets only its own tool
    ),
    "doc-analyzer": AgentDefinition(
        description="Extracts claims with page references from the named documents.",
        prompt="Analyse only the documents named in your task, using load_document. " + FINDING_FORMAT,
        tools=["mcp__research__load_document"],
    ),
    "synthesizer": AgentDefinition(
        description="Combines the findings supplied in its prompt into a report draft. Call only once findings exist.",
        prompt=(
            "Combine the findings in your prompt into a draft. Keep the source_url or document_name and the date "
            "with every claim. If two findings give different values for the same measure, report BOTH with their "
            "sources and dates and mark it as contested. Use three sections: Well-established (agreeing sources), "
            "Contested (conflicting values), Coverage gaps (anything missing or failed, and why)."
        ),
        tools=[],                                             # works only from what it is given
    ),
}
```

Checklist:

- [ ] The format separates content (`claim`, `evidence_excerpt`) from metadata (`source_url`, `document_name`, `page`, `published`).
- [ ] The synthesizer has no tools and no way to look anything up, so its output can only be built from the supplied findings.

### Step 1: the coordinator and explicit context passing

```python
COORDINATOR = """You coordinate a research team and produce a cited report.

1. Decompose the topic into distinct sub-areas that cover its full breadth.
2. Delegate each sub-area to web-searcher (web) or doc-analyzer (named documents). Subagents cannot see this
   conversation: put the sub-topic, the quality criteria and any prior findings they need IN THE TASK PROMPT.
3. When all results are in, call synthesizer ONCE, pasting ALL findings as JSON into its prompt.
4. If a subagent returns an error object, retry with a changed query or proceed with partial results.
   Record the failed sub-area under Coverage gaps.
5. Output the synthesizer's draft as the final report. Do not add claims no finding supports."""


def make_options(extra_prompt: str = "") -> ClaudeAgentOptions:
    return ClaudeAgentOptions(
        system_prompt=COORDINATOR + extra_prompt,
        mcp_servers={"research": server},
        allowed_tools=["Agent", "mcp__research__*"],            # "Agent" is what the exam calls "Task"
        agents=agents,
        max_turns=40,                                           # bounds on a loop the model steers, not the stopping rule
        max_budget_usd=5.0,
    )


async def run(topic: str, options: ClaudeAgentOptions) -> tuple[str, float]:
    start, text = time.perf_counter(), ""
    async for m in query(prompt=f"Research: {topic}", options=options):
        if isinstance(m, ResultMessage) and m.subtype == "success":
            text = m.result
    return text, time.perf_counter() - start
```

Checklist:

- [ ] `"Agent"` is in `allowed_tools`; without it the coordinator cannot delegate at all.
- [ ] Subagents get what they need only through the task prompt they are given: they do not inherit the coordinator's history.
- [ ] The synthesizer's prompt contains the full findings, pasted by the coordinator, not a reference to earlier results.

### Step 2: parallel versus sequential

The coordinator runs subagents in parallel by emitting several delegation calls in one response. Make the difference observable with a controlled prompt and the timing stub.

```python
TOPIC = "AI in music: market size and studio adoption, plus the analyst report in 'report.pdf'"

PARALLEL = "\nIssue all independent delegations in ONE response so they run concurrently."
SEQUENTIAL = "\nDelegate ONE subagent at a time. Wait for its result before delegating the next."

async def compare():
    # Same topic and tools; only the delegation style differs, so the time gap isolates parallelism.
    _, t_par = await run(TOPIC, make_options(PARALLEL))
    _, t_seq = await run(TOPIC, make_options(SEQUENTIAL))
    print(f"parallel {t_par:.1f}s  sequential {t_seq:.1f}s  speed-up x{t_seq / t_par:.1f}")

asyncio.run(compare())
```

To see the number of parallel calls, log each delegation as it happens:

```python
async def log_delegation(input_data, tool_use_id, context):
    print(f"{time.perf_counter():.1f}  delegating -> {input_data['tool_input'].get('subagent_type')}")
    return {}

# add to make_options(...): hooks={"PreToolUse": [HookMatcher(matcher="Agent", hooks=[log_delegation])]}
```

Checklist:

- [ ] In the parallel run the delegation timestamps are almost identical; in the sequential run they are about `LATENCY` apart.
- [ ] Wall-clock time drops roughly in proportion to the number of independent sub-areas.
- [ ] Dependent steps stay sequential: the synthesizer must wait for every finding.

### Step 3 test: provenance survives synthesis

```python
report, _ = asyncio.run(run(TOPIC, make_options(PARALLEL)))
print(report)
```

Checklist:

- [ ] Every claim in the report names its source (URL or document and page) and date.
- [ ] No figure appears without a source, and nothing appears that was not in the findings.

### Step 4: error propagation and partial results

```python
FAIL_QUERIES.add("adoption")        # the studio-adoption search now times out
report, _ = asyncio.run(run(TOPIC, make_options(PARALLEL)))
print(report)
```

The tool returns an error object with `failure_type`, `attempted_query`, `partial_results` and `alternatives`. The subagent passes it up unchanged, and the coordinator decides what to do with it.

Checklist:

- [ ] The coordinator received the failure type and the attempted query, not just "search failed".
- [ ] It tried an alternative (a narrower query or the `doc-analyzer`) or went on with the partial results.
- [ ] The final report lists adoption under **Coverage gaps** with the reason, instead of omitting it.
- [ ] Contrast with two anti-patterns: swallowing the error and returning an empty "success", or aborting the whole run on one timeout. In the empty-result case the report would look complete while missing a section.
- [ ] A legitimate empty result (`"results": []`) is not treated as a failure.

### Step 5: conflicting sources

The `market size` query returns two credible sources with $2.1B and $3.4B.

```python
FAIL_QUERIES.clear()
report, _ = asyncio.run(run("AI in music: market size", make_options(PARALLEL)))
print(report)
```

Expected shape:

```text
Well-established
- (findings with a single or agreeing source, each with source and date)

Contested
- Market size 2024: $2.1B (analyst report, 2025-02-01, example.com) vs $3.4B (industry survey, 2025-05-15, example.org).
  Different methodologies or definitions may explain the gap.

Coverage gaps
- (none)
```

Checklist:

- [ ] Both values appear, each with its source and date. The synthesizer did not pick one, average them, or drop one.
- [ ] The conflict is under **Contested**, and findings with agreeing sources are under **Well-established**.
- [ ] Publication dates are shown, so a difference due to timing is not mistaken for a contradiction.
- [ ] Add a third source that agrees with $2.1B and confirm the report still shows the disagreement rather than moving it to "established".

## Commentary: what each step teaches

Names and flags are illustrative; the exam tests the reasoning, not the exact parameter.

### Step 1: coordinator and explicit context passing

- **Concept:** Subagents run in isolated contexts. Nothing the coordinator knows reaches them unless it is written into the task prompt, so delegation is an explicit hand-off of everything the worker needs.
- **What to observe:** The task prompts the coordinator writes contain the sub-topic, the quality criteria and any prior findings; the synthesizer's prompt contains the full findings as pasted text. A subagent that answers a different question, or asks for information it was never given, indicates the hand-off was incomplete. Without the delegation tool in `allowed_tools`, the coordinator cannot delegate at all and attempts the work itself.
- **Why it matters for the exam:** Builds "subagents do not inherit context; pass it explicitly" and "the coordinator owns decomposition and routing". Also builds least privilege: the synthesizer has no tools.

### Step 2: parallel execution

- **Concept:** Independent sub-tasks have no ordering constraint, so issuing all delegations in one response lets them run concurrently. Tasks that depend on earlier results stay sequential.
- **What to observe:** Delegation timestamps are almost identical in the parallel run and roughly one tool latency apart in the sequential run; wall-clock time falls in proportion to the number of independent sub-areas. If the parallel run is not faster, delegations were emitted one per turn, or the sub-areas were not independent. The synthesizer correctly waits for all findings.
- **Why it matters for the exam:** Builds "parallelise independent work, serialise dependencies" and the idea that latency is a design property of the coordinator, not of the subagents.

### Step 3: structured findings and provenance

- **Concept:** Provenance survives synthesis only if it is carried as data. Separating content (claim, excerpt) from metadata (source, page, date) in a fixed format lets the synthesizer attach attribution mechanically instead of recalling it.
- **What to observe:** Every claim in the report names a source and date, and nothing appears that was not in the findings. A claim without attribution means it was summarised from memory or the metadata was dropped in an earlier hand-off. Averaged or merged figures indicate the instruction against combining differing values was missing.
- **Why it matters for the exam:** Builds "provenance must survive synthesis" and "tools-free synthesizer works only from what it is given".

### Step 4: error propagation

- **Concept:** An error crossing an agent boundary carries context: what failed, what was attempted, what partial results exist, and what alternatives remain. The component with the widest view (the coordinator) decides whether to retry, substitute or proceed.
- **What to observe:** With the adoption search failing, the coordinator sees the failure type and query, tries an alternative or continues, and the report lists adoption under Coverage gaps with the reason. A report that looks complete but lacks adoption means the failure was converted into an empty success. A run that stops entirely means one local fault was escalated into a global one. A genuinely empty result must not trigger any of this.
- **Why it matters for the exam:** Builds "structured errors enable recovery" and "distinguish could-not-search from found-nothing" (Question 8 in the [multi-agent scenario](../../scenarios/03-multi-agent-research-system/README.md)).

### Step 5: conflicting sources

- **Concept:** Disagreement between credible sources is information. Synthesis keeps both values with sources and dates, and the report structure separates agreed findings from contested ones.
- **What to observe:** Both $2.1B and $3.4B appear with source and date under Contested. If one value is chosen, averaged or dropped, the synthesizer prompt lacks the keep-both rule. Adding a third agreeing source should leave the disagreement visible; if it silently becomes "established", the synthesis resolved the conflict by majority.
- **Why it matters for the exam:** Builds "preserve conflicts rather than resolve them silently" and the use of dates and methodology to explain differences.

### Common mistakes

- Relying on automatic context inheritance for subagents: workers lack the topic details and return generic or off-target findings.
- Returning "search failed" with no type, query or partial results: the coordinator cannot choose between retrying, rewording and proceeding.
- Marking a failed search as an empty success: the report is silently incomplete and looks authoritative.
- Terminating the workflow on the first subagent failure: partial progress is discarded for a recoverable fault.
- Delegating one subagent per turn when the tasks are independent: latency scales with the number of sub-areas.
- Giving the synthesizer web tools "just in case": roles blur, it re-researches instead of synthesising, and attribution to the collected findings is lost.

### Extension ideas

- Make the topic broad enough that decomposition is the failure point (a narrow set of sub-areas leaves large parts uncovered), then trace the incomplete report to its first faulty stage. Tests diagnosing at the coordinator rather than blaming downstream workers (Question 7).
- Add a lightweight verification need for the synthesizer, such as checking a single date, and decide between a scoped tool on the synthesizer and routing through the coordinator. Tests least privilege versus coordination overhead (Question 9).
- Make two findings differ because they measure different things (global versus regional market) and extend the finding format so the synthesizer can tell a real conflict from a definitional difference. Tests metadata design for correct interpretation.
