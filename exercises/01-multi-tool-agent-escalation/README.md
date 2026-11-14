# Exercise 1: Build a Multi-Tool Agent with Escalation Logic

## Objective

Practice designing an agentic loop with tool integration, structured error handling, and escalation patterns.

## Domains reinforced

[1](../../domains/01-agentic-architecture-orchestration/README.md), [2](../../domains/02-tool-design-mcp-integration/README.md), [5](../../domains/05-context-management-reliability/README.md)

## Steps

1. Define 3-4 MCP tools with detailed descriptions that clearly differentiate each tool's purpose, expected inputs, and boundary conditions. Include at least two tools with similar functionality that require careful description to avoid selection confusion.
2. Implement an agentic loop that checks `stop_reason` to determine whether to continue tool execution or present the final response. Handle both `"tool_use"` and `"end_turn"` correctly.
3. Add structured error responses to your tools: `errorCategory` (transient/validation/permission), `isRetryable` boolean, and human-readable descriptions. Test that the agent handles each error type appropriately (retrying transient errors, explaining business errors to the user).
4. Implement a programmatic hook that intercepts tool calls to enforce a business rule (e.g., blocking operations above a threshold amount), redirecting to an escalation workflow when triggered.
5. Test with multi-concern messages and verify the agent decomposes the request, handles each concern, and synthesizes a unified response.

## Worked solution

The exam guide says `stop_reason`, so this uses the raw Messages API loop rather than the Agent SDK, which runs the loop for you. The same design in the Agent SDK is in the [customer support scenario](../../scenarios/01-customer-support-resolution-agent/README.md). Background: [1.1](../../domains/01-agentic-architecture-orchestration/1.1-agentic-loops.md), [1.5](../../domains/01-agentic-architecture-orchestration/1.5-hooks-interception-normalization.md), [2.1](../../domains/02-tool-design-mcp-integration/2.1-tool-interface-design.md), [2.2](../../domains/02-tool-design-mcp-integration/2.2-structured-error-responses.md).

### Step 1: tool definitions

`lookup_order` and `list_customer_orders` overlap. Their descriptions state the input each takes, what each returns, and when to use the other.

```python
# The description is the model's only basis for choosing between overlapping tools.
TOOLS = [
    {
        "name": "get_customer",
        "description": (
            "Find a customer by email or phone and return customer_id, name and verification status. "
            "Call this FIRST in every conversation; the other tools need the customer_id it returns. "
            "Does not return orders. If several customers match, returns all of them: ask the user to disambiguate."
        ),
        "input_schema": {
            "type": "object",
            "properties": {"email": {"type": "string"}, "phone": {"type": "string"}},
        },
    },
    {
        "name": "lookup_order",
        "description": (
            "Return full details of ONE order (items, total, status, delivery date) when you already have "
            "its order_id, such as 'ORD-1234'. Requires the order_id; do not guess one. "
            "If you only know the customer, use list_customer_orders instead."
        ),
        "input_schema": {
            "type": "object",
            "properties": {"order_id": {"type": "string", "description": "Format ORD-####"}},
            "required": ["order_id"],
        },
    },
    {
        "name": "list_customer_orders",
        "description": (
            "Return a short summary (order_id, date, status) of a customer's recent orders, newest first, "
            "when you have a customer_id but no order_id. Does NOT return line items or totals; "
            "call lookup_order for those."
        ),
        "input_schema": {
            "type": "object",
            "properties": {"customer_id": {"type": "string"}, "limit": {"type": "integer", "default": 5}},
            "required": ["customer_id"],
        },
    },
    # The description only tells the model what to expect; the limit itself is enforced in code (step 4).
    {
        "name": "process_refund",
        "description": (
            "Refund an order the customer is entitled to. Requires a verified customer_id and an order_id "
            "that belongs to that customer. Amounts above the auto-approval limit are routed to a human "
            "automatically; you do not need to check the limit yourself. Cannot refund an order twice."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "customer_id": {"type": "string"},
                "order_id": {"type": "string"},
                "amount": {"type": "number"},
                "reason": {"type": "string"},
            },
            "required": ["customer_id", "order_id", "amount", "reason"],
        },
    },
    {
        "name": "escalate_to_human",
        "description": (
            "Hand the case to a human agent with a summary. Use when the customer explicitly asks for a human, "
            "the request is outside policy (exceptions, price matching), or you cannot make progress. "
            "Do NOT use for a normal refund the tools can handle."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "customer_id": {"type": ["string", "null"]},
                "summary": {"type": "string", "description": "Issue, what was tried, what is needed"},
                "recommended_action": {"type": "string"},
            },
            "required": ["summary"],
        },
    },
]
```

Checklist:

- [ ] Each description says what the tool returns, what it needs, and which neighbouring tool to use otherwise.
- [ ] Asking "what did order ORD-1 cost?" selects `lookup_order`; "what have I bought lately?" selects `list_customer_orders`.

### Step 3 first: structured errors

The backend is a stub with a transient failure injected so you can test each error category.

```python
import json
import random

CUSTOMERS = {"c1": {"name": "Ada", "email": "ada@example.com", "verified": True}}
ORDERS = {
    "ORD-1": {"customer_id": "c1", "total": 80.0, "status": "delivered", "refunded": False},
    "ORD-2": {"customer_id": "c1", "total": 899.0, "status": "delivered", "refunded": False},
}


def err(category: str, retryable: bool, message: str) -> dict:
    # One error shape everywhere, so the model decides retry vs explain from fields, not from parsing prose.
    return {"is_error": True, "content": json.dumps(
        {"errorCategory": category, "isRetryable": retryable, "message": message})}


def ok(payload) -> dict:
    return {"is_error": False, "content": json.dumps(payload)}


def run_tool(name: str, args: dict) -> dict:
    if name == "get_customer":
        hits = [{"customer_id": cid, **{k: v for k, v in c.items() if k != "email"}}
                for cid, c in CUSTOMERS.items() if c["email"] == args.get("email")]
        return ok({"matches": hits})                                  # zero matches is a valid result, not an error

    if name == "lookup_order":
        if random.random() < 0.3:                                     # inject a transient failure
            return err("transient", True, "Order service timed out. Retry the same call.")
        order = ORDERS.get(args["order_id"])
        if not order:
            return err("validation", False, f"No order {args['order_id']}. Ask the customer to check the number.")
        return ok({"order_id": args["order_id"], **order})

    if name == "list_customer_orders":
        return ok([{"order_id": oid, "status": o["status"]} for oid, o in ORDERS.items()
                   if o["customer_id"] == args["customer_id"]][: args.get("limit", 5)])

    if name == "process_refund":
        order = ORDERS.get(args["order_id"])
        if not order or order["customer_id"] != args["customer_id"]:
            return err("validation", False, "Order not found for this customer.")
        if order["refunded"]:
            return err("business                                      # state change makes the duplicate-refund case testable", False, "This order was already refunded. Explain this to the customer; do not retry.")
        order["refunded"] = True
        return ok({"refund_id": "R-1", "amount": args["amount"]})

    if name == "escalate_to_human":
        return ok({"ticket": "T-100", "status": "queued"})

    return err("validation", False, f"Unknown tool {name}")
```

Checklist:

- [ ] A failure carries `errorCategory`, `isRetryable` and a message the model can relay.
- [ ] Zero matches or an empty list is returned as a normal result with `is_error: False`.
- [ ] The `business` category is an addition for policy violations: the model explains them, it does not retry.

### Step 2: the agentic loop

```python
import anthropic

client = anthropic.Anthropic()
MODEL = "claude-sonnet-5-5"

SYSTEM = """You are a support agent.
- Verify the customer with get_customer before any order or refund action.
- Break a message with several concerns into separate items, handle each, then reply once covering all of them.
- On an error result: if isRetryable is true, retry the same call (at most twice). If false, explain the
  problem plainly to the customer and do not retry. Never expose raw error JSON.
- Escalate with escalate_to_human when the customer asks for a human or the request is outside policy."""


def run_agent(user_message: str, messages: list | None = None, max_iterations: int = 15) -> tuple[str, list]:
    messages = messages or []
    messages.append({"role": "user", "content": user_message})

    for _ in range(max_iterations):                         # safety cap only; stop_reason ends the loop
        response = client.messages.create(
            model=MODEL, max_tokens=2048, system=SYSTEM, tools=TOOLS, messages=messages,
        )
        messages.append({"role": "assistant", "content": response.content})   # full content, including tool_use blocks

        if response.stop_reason == "end_turn":
            return "".join(b.text for b in response.content if b.type == "text"), messages

        if response.stop_reason == "tool_use":
            results = []
            for block in response.content:
                if block.type == "tool_use":
                    # Every call passes through this one function, which is where step 4 hooks in.
                    out = guarded_run_tool(block.name, block.input)          # step 4 wraps run_tool
                    results.append({"type": "tool_result", "tool_use_id": block.id,
                                    "content": out["content"], "is_error": out["is_error"]})
            messages.append({"role": "user", "content": results})            # all results in one user message
            continue

        raise RuntimeError(f"Unexpected stop_reason: {response.stop_reason}")   # e.g. max_tokens, refusal
    raise RuntimeError("Iteration cap reached")


def guarded_run_tool(name: str, args: dict) -> dict:     # replaced in step 4
    return run_tool(name, args)
```

Checklist:

- [ ] The loop continues on `"tool_use"` and stops on `"end_turn"`, not on "the text looks finished" or a turn count.
- [ ] The assistant message is appended with its full `content`, and tool results go back as `tool_result` blocks matched by `tool_use_id`.
- [ ] Parallel `tool_use` blocks in one response are all answered in one user message.

### Step 3 tests: each error type

```python
# transient: run several times; lookup_order fails ~30%, the agent should retry silently
print(run_agent("Customer ada@example.com. What is the status of ORD-1?")[0])

# validation: the agent should explain the order number is unknown and ask for a correct one
print(run_agent("Customer ada@example.com. Where is order ORD-999?")[0])

# business: refund the same order twice in one conversation
_, history = run_agent("I'm ada@example.com. Refund ORD-1 ($80), it arrived damaged.")
print(run_agent("Please refund it again.", history)[0])    # explained, not retried
```

Checklist:

- [ ] Transient errors are retried and do not reach the customer.
- [ ] Validation and business errors are explained in plain language and are not retried.
- [ ] Add a `permission` case yourself (e.g. an unverified customer) and confirm the agent explains rather than retries.

### Step 4: a hook that enforces a business rule

A hook runs in code, so it holds even if the model ignores the prompt. This one blocks refunds above $500 and starts the escalation instead.

```python
REFUND_LIMIT = 500.0


def guarded_run_tool(name: str, args: dict) -> dict:
    # Runs after the model decides and before the backend acts, so the prompt cannot bypass it.
    if name == "process_refund" and args.get("amount", 0) > REFUND_LIMIT:
        ticket = run_tool("escalate_to_human", {
            "customer_id": args.get("customer_id"),
            "summary": f"Refund of ${args['amount']:.2f} on {args.get('order_id')} exceeds the ${REFUND_LIMIT:.0f} limit. "
                       f"Reason given: {args.get('reason')}",
            "recommended_action": "Review and approve or deny the refund",
        })
        return err("permission", False,   # not retryable: a retry would hit the same rule, so the model should explain instead
                   f"Refunds above ${REFUND_LIMIT:.0f} need human approval. The case was escalated "
                   f"({json.loads(ticket['content'])['ticket']}). Tell the customer a specialist will follow up.")
    return run_tool(name, args)
```

Checklist:

- [ ] `process_refund` for ORD-2 ($899) never reaches the backend: `ORDERS["ORD-2"]["refunded"]` stays `False`.
- [ ] The customer is told a human will follow up, not that the refund failed.
- [ ] Refunding ORD-1 ($80) still goes straight through.

### Step 5: a multi-concern message

```python
text, history = run_agent(
    "Hi, I'm ada@example.com. Two things: ORD-1 arrived damaged and I want a refund, "
    "and can you tell me where ORD-2 is?"
)
print(text)
for m in history:
    if m["role"] == "assistant":
        print([b.name for b in m["content"] if b.type == "tool_use"])     # the tool sequence shows the decomposition
```

Expected tool calls: `get_customer`, then `lookup_order` for both orders (possibly in parallel), then `process_refund` for ORD-1, then one combined answer.

Checklist:

- [ ] Both concerns are addressed in a single final message.
- [ ] Identity is verified once, before the order actions.
- [ ] Change the message to include a $899 refund on ORD-2: the hook escalates it while the ORD-1 refund still succeeds.

## Commentary: what each step teaches

Names and flags are illustrative; the exam tests the reasoning, not the exact parameter.

### Step 1: tool definitions

- **Concept:** The model chooses between tools using only their names, descriptions and schemas. When two tools overlap, the description is the routing logic, so it must state input, output, and the neighbouring tool to use instead.
- **What to observe:** "What did ORD-1 cost?" selects `lookup_order` and "what have I bought lately?" selects `list_customer_orders` without any extra prompt text. If the wrong tool is picked, read the two descriptions side by side: a missing boundary ("does NOT return line items") or an input requirement left implicit is the usual cause, not a need for more tools or examples.
- **Why it matters for the exam:** Builds the rule "tool descriptions drive selection". Fix descriptions before adding few-shot examples, keyword routing or merging tools (the Question 2 distractors in the [customer support scenario](../../scenarios/01-customer-support-resolution-agent/README.md)).

### Step 2: the agentic loop

- **Concept:** The loop is driven by the model's own signal. `tool_use` means execute and continue; `end_turn` means the answer is complete. The iteration cap is a safety net, not the stopping rule.
- **What to observe:** The tool sequence in the history varies between runs (a transient retry adds a turn) while the loop still terminates correctly. A conversation that ends early with tool calls unanswered points at a missing `tool_result` match by `tool_use_id`; one that never ends usually means the stop condition was inferred from text or turn count.
- **Why it matters for the exam:** Builds "the model decides the path, the loop only executes". Parsing the assistant text for completion, or fixing the number of iterations, are the anti-patterns the exam names.

### Step 3: structured errors

- **Concept:** An error crossing the tool boundary is an input to the model's next decision. It needs a category, a retryability flag and a message the model can relay, so recovery is chosen from data rather than guessed.
- **What to observe:** Transient failures are retried and never mentioned to the customer; validation and business failures are explained once and not retried. If the agent retries a business error, the `isRetryable: false` flag or the system prompt rule is not reaching the model. If it apologises for a transient failure, the retry instruction is missing. Zero matches must come back as success with an empty list, otherwise the agent reports a fault where there is none.
- **Why it matters for the exam:** Builds "structured errors enable recovery" and the distinction between "could not search" and "searched, found nothing".

### Step 4: the enforcement hook

- **Concept:** A rule with financial or legal consequences must hold on every run, so it is enforced in code between the model's decision and the backend action. The hook redirects to a legitimate workflow (escalation) instead of only refusing.
- **What to observe:** The $899 refund never changes `refunded`, an escalation ticket exists, and the customer is told a human will follow up. Run it repeatedly, and with a system prompt that omits the limit: the outcome must not change. If the refund goes through, the check is in the prompt rather than in `guarded_run_tool`, or the hook is not on the path every tool call takes.
- **Why it matters for the exam:** Builds "enforce in code what must always hold". Prompt wording and few-shot examples shift likelihood but cannot guarantee compliance (Question 1 in the scenario).

### Step 5: multi-concern messages

- **Concept:** One message can contain several independent tasks. The agent decomposes them, shares prerequisites (verify the customer once), handles each, and merges the results into one reply.
- **What to observe:** One `get_customer`, then order calls (possibly parallel), then a refund, then a single answer covering both concerns. Verification repeated per concern shows the prerequisite is not stated in the system prompt; a reply that covers only one concern shows the decomposition instruction is missing. In the $899 variant, the escalation must not block the $80 refund.
- **Why it matters for the exam:** Builds the link between loop mechanics, tool ordering and enforcement: each rule sits at the layer that can guarantee it.

### Common mistakes

- Ending the loop when the reply contains text, or after N iterations: truncated answers when a tool call and text appear together, and unanswered tool calls.
- Returning every failure as a plain string such as "error": the model cannot separate retryable from permanent failures and either hammers the service or gives up early.
- Returning a failed lookup as success with an empty body: the agent presents a fabricated or empty answer because no failure was signalled.
- Putting the $500 limit only in the system prompt: usually followed, occasionally not, and the failures are exactly the expensive ones.
- Giving the escalation tool a vague description: the model escalates routine refunds, or fails to escalate when the customer asks for a human.
- Sending the assistant turn back without its `tool_use` blocks, or splitting parallel results across messages: the next call is rejected or the conversation state is inconsistent.

### Extension ideas

- Add a prerequisite gate in the hook so `process_refund` is blocked until `get_customer` has returned a verified customer in the same conversation, and compare with enforcing it through the prompt. Tests programmatic prerequisites versus probabilistic ordering.
- Add a `PostToolUse`-style normalisation step so one tool returns Unix timestamps and another ISO strings, and confirm the model compares dates correctly only after normalisation. Tests deterministic data cleanup before the model sees results.
- Replace the fixed $500 limit with a per-customer tier looked up in code, and add an escalation payload (customer, root cause, recommended action) that a human could act on without the transcript. Tests handoff design for a reader with no context.
