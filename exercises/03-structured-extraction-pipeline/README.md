# Exercise 3: Build a Structured Data Extraction Pipeline

## Objective

Practice designing JSON schemas, using `tool_use` for structured output, implementing validation-retry loops, and designing batch processing strategies.

## Domains reinforced

[4](../../domains/04-prompt-engineering-structured-output/README.md), [5](../../domains/05-context-management-reliability/README.md)

## Steps

1. Define an extraction tool with a JSON schema containing required and optional fields, an enum with an "other" + detail string pattern, and nullable fields for information that may not exist in source documents. Process documents where some fields are absent and verify the model returns null rather than fabricating values.
2. Implement a validation-retry loop: when Pydantic or JSON schema validation fails, send a follow-up request including the document, the failed extraction, and the specific validation error. Track which errors are resolvable via retry (format mismatches) versus which are not (information absent from source).
3. Add few-shot examples demonstrating extraction from documents with varied formats (e.g., inline citations vs bibliographies, narrative descriptions vs structured tables) and verify improved handling of structural variety.
4. Design a batch processing strategy: submit a batch of 100 documents using the Message Batches API, handle failures by `custom_id`, resubmit failed documents with modifications (e.g., chunking oversized documents), and calculate total processing time relative to SLA constraints.
5. Implement a human review routing strategy: have the model output field-level confidence scores, route low-confidence extractions to human review, and analyze accuracy by document type and field to verify consistent performance.

## Worked solution

The documents here are research papers and reports, so the "varied formats" in step 3 are real. The invoice version of the same pipeline is in the [extraction scenario](../../scenarios/06-structured-data-extraction/README.md). Background: [4.2](../../domains/04-prompt-engineering-structured-output/4.2-few-shot-prompting.md), [4.3](../../domains/04-prompt-engineering-structured-output/4.3-structured-output-tool-use.md), [4.4](../../domains/04-prompt-engineering-structured-output/4.4-validation-retry-loops.md), [4.5](../../domains/04-prompt-engineering-structured-output/4.5-batch-processing.md), [5.5](../../domains/05-context-management-reliability/5.5-human-review-confidence.md).

Build a small test set of at least 10 documents first, with a mix of: complete papers, papers with no publication date, papers with no abstract, a report with a table of results, and one that cites sources inline rather than in a bibliography.

### Step 1: schema and absent fields

```python
import anthropic

client = anthropic.Anthropic()
MODEL = "claude-sonnet-5-5"     # forced tool_choice is rejected on some newer models; see 2.3 and 4.3

EXTRACT_DOC = {
    "name": "extract_document",
    "description": "Record metadata for a research paper or report. Use null for anything the document does not state.",
    "input_schema": {
        "type": "object",
        "properties": {
            "title": {"type": "string"},                                                      # required
            "authors": {"type": "array", "items": {"type": "string"}},                        # required
            "publication_date": {"type": ["string", "null"], "description": "ISO 8601; null if not stated"},
            "abstract": {"type": ["string", "null"]},
            "doc_type": {"type": "string", "enum": ["journal_article", "report", "preprint", "unclear", "other"]},
            "doc_type_detail": {"type": ["string", "null"], "description": "Required when doc_type is 'other'"},
            "citation_style": {"type": "string", "enum": ["bibliography", "inline", "none"]},
            "citations": {"type": "array", "items": {"type": "string"}},
            "key_results": {"type": "array", "items": {"type": "object", "properties": {
                "metric": {"type": "string"}, "value": {"type": ["number", "null"]}, "unit": {"type": ["string", "null"]}},
                "required": ["metric"]}},
            "confidence": {"type": "object", "description": "0-1 per field",   # self-reported: usable for routing only after calibration (step 5)
                           "properties": {"title": {"type": "number"}, "authors": {"type": "number"},
                                          "publication_date": {"type": "number"}, "doc_type": {"type": "number"}}},
        },
        "required": ["title", "authors", "doc_type", "citation_style", "citations"],
    },
}
CHOICE = {"type": "tool", "name": "extract_document"}     # forces schema-shaped output: guarantees syntax, not that values are right


def extract_raw(document: str, system: str = "Extract metadata with the extract_document tool. Never guess a missing field.") -> dict:
    r = client.messages.create(
        model=MODEL, max_tokens=2048, system=system, tools=[EXTRACT_DOC], tool_choice=CHOICE,
        messages=[{"role": "user", "content": f"<document>\n{document}\n</document>"}],
    )
    return next(b.input for b in r.content if b.type == "tool_use")


# absent fields
no_date = "Title: Sleep and memory\nAuthors: R. Chen\nAbstract: We study ...\n(no date anywhere)"
out = extract_raw(no_date)
assert out["publication_date"] is None, out        # null, not an invented date
```

Checklist:

- [ ] `publication_date`, `abstract` and per-metric `value` are nullable, so the model can say "not stated" instead of inventing something.
- [ ] `doc_type` has `unclear` and `other` + `doc_type_detail`, so odd inputs do not get forced into a wrong category.
- [ ] Tool use guarantees the JSON syntax; it does not guarantee the values are correct. That is step 2.
- [ ] Run the absent-field documents and confirm every missing field is `null`.

### Step 2: validation and retry with error feedback

```python
import json
from datetime import date
from typing import Literal
from pydantic import BaseModel, ValidationError, model_validator


class KeyResult(BaseModel):
    metric: str
    value: float | None = None
    unit: str | None = None


class Doc(BaseModel):
    title: str
    authors: list[str]
    publication_date: str | None = None
    abstract: str | None = None
    doc_type: Literal["journal_article", "report", "preprint", "unclear", "other"]
    doc_type_detail: str | None = None
    citation_style: Literal["bibliography", "inline", "none"]
    citations: list[str]
    key_results: list[KeyResult] = []
    confidence: dict[str, float] = {}

    @model_validator(mode="after")
    def checks(self):    # semantic rules the JSON schema cannot express; tool use alone never catches these
        if self.publication_date is not None:
            date.fromisoformat(self.publication_date)                    # format error: fixable by retry
        if self.doc_type == "other" and not self.doc_type_detail:
            raise ValueError("doc_type is 'other' but doc_type_detail is empty")
        if self.citation_style == "none" and self.citations:
            raise ValueError("citation_style is 'none' but citations were listed")
        return self


RETRYABLE = {"value_error", "string_pattern_mismatch", "literal_error", "enum"}      # the answer is in the document
NOT_RETRYABLE = {"missing", "string_type", "list_type"}                                # required field absent or null: not in the document


def extract_with_retry(document: str, max_retries: int = 2) -> dict:
    messages = [{"role": "user", "content": f"<document>\n{document}\n</document>"}]
    stats = {"retries": 0}
    for attempt in range(max_retries + 1):
        r = client.messages.create(model=MODEL, max_tokens=2048, tools=[EXTRACT_DOC], tool_choice=CHOICE,
                                   system="Extract metadata with the extract_document tool. Never guess a missing field.",
                                   messages=messages)
        tool_use = next(b for b in r.content if b.type == "tool_use")
        raw = tool_use.input
        try:
            return {"status": "ok", "doc": Doc.model_validate(raw), **stats}
        except ValidationError as e:
            errors = e.errors()
            if all(err["type"] in NOT_RETRYABLE for err in errors) or attempt == max_retries:
                return {"status": "needs_review", "raw": raw, "errors": errors, **stats}   # a retry cannot invent the data
            stats["retries"] += 1
            messages += [
                # the retry carries the document, the failed extraction and the specific error
                {"role": "assistant", "content": [{"type": "tool_use", "id": tool_use.id,
                                                   "name": "extract_document", "input": raw}]},
                {"role": "user", "content": [{"type": "tool_result", "tool_use_id": tool_use.id, "is_error": True,
                                              "content": f"Validation failed: {e}\nRe-read the document and fix only the "
                                                         "failing fields. If it is not in the document, use null."}]},
            ]
```

Test the two classes:

```python
bad_format = "Title: X\nAuthors: Y\nPublished 3rd of March 2024 ..."     # model may emit '3 March 2024', not ISO
absent = "Title: X\n(no authors listed anywhere)"                         # nothing to retry toward
for doc in (bad_format, absent):
    res = extract_with_retry(doc)
    print(res["status"], res["retries"], res.get("errors"))
```

Checklist:

- [ ] A format mismatch (date not ISO) is fixed on the second attempt after the error is fed back.
- [ ] A document that simply lacks the information is sent to review, not retried until the cap.
- [ ] Record retry rate per error type: errors that retry successfully are format fixes; ones that never do point at missing source data or a schema problem.

### Step 3: few-shot for varied formats

```python
FEW_SHOT_SYSTEM = """Extract metadata with the extract_document tool. Never guess a missing field; use null.

<example>
Input: a paper ending with a numbered "References" list.
Output: citation_style "bibliography"; citations is the list of reference entries.
</example>
<example>
Input: a blog-style report with no references list that says "as shown by Smith et al. (2021) ...".
Output: citation_style "inline"; citations contains "Smith et al. (2021)" for each mention.
   Do not invent a bibliography entry that is not in the text.
</example>
<example>
Input: a results table with rows "Accuracy | 91.2 | %" and prose saying "latency was about 40 ms".
Output: key_results [{metric "accuracy", value 91.2, unit "%"}, {metric "latency", value 40, unit "ms"}].
   "about" is still a stated number; "significantly lower" with no number gives value null.
</example>
<example>
Input: a document with no date.
Output: publication_date null, confidence.publication_date low. Not a guess from the context.
</example>"""

# compare against the zero-shot baseline on the same documents
for doc in test_docs:
    zero = extract_raw(doc)
    few = extract_raw(doc, system=FEW_SHOT_SYSTEM)
```

Measure the change on the inline-citation and table documents, which are the ones the examples target:

```python
def score(extractions, labels):
    return sum(e["citation_style"] == l["citation_style"] for e, l in zip(extractions, labels)) / len(labels)
```

Checklist:

- [ ] Each example shows the reasoning (why inline, why null) as well as the output, so it generalises.
- [ ] Citation style accuracy and key-result extraction on the varied-format documents improve against zero-shot.
- [ ] The examples cover absence as well as presence, so few-shot does not push the model toward filling every field.

### Step 4: batch of 100 documents

```python
import time
from anthropic.types.message_create_params import MessageCreateParamsNonStreaming
from anthropic.types.messages.batch_create_params import Request


def build(doc_id: str, text: str) -> Request:
    return Request(
        custom_id=f"doc-{doc_id}",                                   # correlates results back to inputs
        params=MessageCreateParamsNonStreaming(
            model=MODEL, max_tokens=2048, system=FEW_SHOT_SYSTEM, tools=[EXTRACT_DOC], tool_choice=CHOICE,
            messages=[{"role": "user", "content": f"<document>\n{text}\n</document>"}]),
    )


def run_batch(docs: dict[str, str], poll_seconds: int = 60) -> tuple[dict, dict]:
    batch = client.messages.batches.create(requests=[build(i, t) for i, t in docs.items()])
    while (batch := client.messages.batches.retrieve(batch.id)).processing_status != "ended":
        time.sleep(poll_seconds)                                     # no latency guarantee, so nothing user-facing may wait here

    done, failed = {}, {}
    for r in client.messages.batches.results(batch.id):
        doc_id = r.custom_id.removeprefix("doc-")
        if r.result.type == "succeeded":
            done[doc_id] = next(b.input for b in r.result.message.content if b.type == "tool_use")
        else:
            failed[doc_id] = r.result.type                           # errored, expired, canceled
    return done, failed


def chunk(text: str, size: int = 40_000) -> list[str]:
    return [text[i:i + size] for i in range(0, len(text), size)]


docs = {str(i): load(i) for i in range(100)}
done, failed = run_batch(docs)

retry_docs = {}
for doc_id, reason in failed.items():
    for n, part in enumerate(chunk(docs[doc_id])):
        retry_docs[f"{doc_id}-part{n}"] = part                       # oversized documents come back as smaller ones
done2, failed2 = run_batch(retry_docs) if retry_docs else ({}, {})
```

SLA arithmetic. Batches finish within 24 hours (most much sooner) and there are no latency guarantees, so work from the worst case:

```python
def max_wait_before_submit(sla_hours: float, batch_window_hours: float = 24, resubmit_rounds: int = 0) -> float:
    """How long documents may sit before being submitted and still meet the SLA."""
    return sla_hours - batch_window_hours * (1 + resubmit_rounds)

print(max_wait_before_submit(30))                      # 6 h: submit at least every 6 h (4 h leaves margin)
print(max_wait_before_submit(30, resubmit_rounds=1))   # negative: a resubmit round cannot fit a 30 h SLA
```

Checklist:

- [ ] Every result is matched to its input by `custom_id`, because batch results are not returned in order.
- [ ] Only failed documents are resubmitted, and with a change (chunking). Resubmitting unchanged repeats the failure.
- [ ] The SLA calculation says whether a retry round fits. For a 30 hour SLA it does not, so failures should be handled with the synchronous `extract_with_retry` instead.
- [ ] The batch is for non-blocking work; it is the wrong tool for a user waiting on a result.
- [ ] Tune the schema and prompt on about 20 documents synchronously before submitting 100, so a prompt bug does not cost a full batch.

### Step 5: confidence routing and accuracy by segment

```python
THRESHOLDS = {"title": 0.90, "authors": 0.90, "publication_date": 0.85, "doc_type": 0.85}   # starting values; replace from the calibration table below


def route(result: dict) -> str:
    if result["status"] != "ok":
        return "human_review"
    d: Doc = result["doc"]
    if d.doc_type == "unclear":
        return "human_review"                         # the model flagging ambiguity is itself a routing signal
    if any(d.confidence.get(f, 0.0) < floor for f, floor in THRESHOLDS.items()):    # a missing score counts as low, not as high
        return "human_review"
    return "auto_accept"
```

Calibrate the thresholds on labelled data and check consistency across segments before reducing review:

```python
import pandas as pd

df = pd.read_csv("labelled_validation.csv")      # doc_type, field, confidence, correct (0/1)
print(df.groupby(["doc_type", "field"]).correct.agg(["mean", "count"]).sort_values("mean").head(10))

for floor in (0.7, 0.8, 0.9, 0.95):
    kept = df[df.confidence >= floor]
    print(floor, f"auto-accepted {len(kept)/len(df):.0%}, accuracy {kept.correct.mean():.1%}")
```

Checklist:

- [ ] Choose the threshold from the accuracy-versus-volume table, not from intuition: the model's confidence is only useful if it was calibrated against labelled data.
- [ ] No `(doc_type, field)` segment is far below the overall accuracy. A 97% average can hide a document type at 70%.
- [ ] Keep a stratified random sample of auto-accepted output going to reviewers to catch drift (see `audit_sample` in the scenario).

## Commentary: what each step teaches

Names and flags are illustrative; the exam tests the reasoning, not the exact parameter.

### Step 1: schema and absent fields

- **Concept:** Schema design shapes model behaviour. Nullable fields give the model a legitimate way to say "not stated"; `unclear` and `other` + detail give it an exit for inputs outside the enumeration. Required fields without such exits push the model to fabricate.
- **What to observe:** Documents with no date or abstract return `null` for exactly those fields. If a value is invented, the field is probably required or non-nullable, or the instruction to use null is missing. If an odd document type is forced into a wrong category, the enum lacks `other`/`unclear`.
- **Why it matters for the exam:** Builds "tool use gives syntax, not semantics". Schema-valid output can still be wrong, which is why step 2 exists. It also builds "design the schema so honesty is possible".

### Step 2: validation and retry

- **Concept:** A retry is useful only when the correct answer is present in the source and the first attempt failed to express it. Feeding back the document, the failed output and the specific error lets the model repair a format mismatch. If the information is absent, no retry can create it.
- **What to observe:** The non-ISO date is fixed on attempt two; the document with no authors goes straight to review without burning the retry cap. Retry rate by error type is the diagnostic: types that retry successfully are format problems; types that never do point at missing data or a schema that demands something the documents lack.
- **Why it matters for the exam:** Builds "classify errors before retrying" and "feed back the specific error, not a generic 'try again'". Unbounded retries on absent data are a named anti-pattern.

### Step 3: few-shot for varied formats

- **Concept:** Few-shot examples teach judgement on ambiguous cases, such as inline versus bibliographic citations, and show how to handle absence. Examples that include the reasoning generalise; examples that show only output are copied.
- **What to observe:** Compared with the zero-shot baseline on the same documents, citation style and table-derived results improve on the varied-format cases. If accuracy does not move, the examples do not match the failing cases. If fields start being filled where they should be null, the examples covered presence only.
- **Why it matters for the exam:** Builds "examples for ambiguity, criteria for rules" and "measure against a baseline before keeping a prompt change". Include negative or absence examples to avoid over-extraction.

### Step 4: batch processing

- **Concept:** The Message Batches API trades latency for cost and suits work nobody is waiting on. Results are correlated by `custom_id`, failures are handled individually, and a resubmission must change something or it fails the same way.
- **What to observe:** Results arrive unordered and are matched by id; only failed documents return, chunked. The SLA arithmetic shows whether a second round fits: with a 24-hour processing window and no latency guarantee, a 30-hour SLA allows no resubmit round, so failures need a synchronous path. Prompt bugs found only after submitting 100 documents show iteration was skipped on a small synchronous sample.
- **Why it matters for the exam:** Builds "batch for non-blocking work, synchronous for blocking work" and "plan from the worst case, not the typical case".

### Step 5: confidence routing

- **Concept:** Human review is a scarce resource, so route by estimated risk. Self-reported confidence is a signal that must be calibrated against labelled data, and aggregate accuracy must be checked per segment.
- **What to observe:** The accuracy-versus-volume table shows what each threshold costs in review load. Some `(doc_type, field)` segment is usually well below the average. If confidence is high but accuracy low, the scores are uncalibrated and thresholds set by intuition are meaningless.
- **Why it matters for the exam:** Builds "calibrate before automating" and "an average can hide a failing segment". A continuing stratified sample of auto-accepted output detects drift.

### Common mistakes

- Making every field required and non-nullable: the model fills gaps with plausible values, and the output still validates.
- Retrying every validation failure up to a cap: absent data is retried pointlessly, cost rises, and the final result is the same.
- Sending a bare "try again" on retry, without the document and the specific error: the model repeats the same mistake.
- Resubmitting failed batch items unchanged: oversized documents fail again. Resubmit with chunking or another modification.
- Choosing the confidence threshold by feel and reporting overall accuracy only: a weak document type or field is hidden by the average and bad data is auto-accepted.
- Using the batch API for a result a user is waiting on: no latency guarantee, so the user's request may sit for hours.

### Extension ideas

- Add a cross-field check, such as `key_results` values that must be consistent with the stated abstract, and decide which failures are retryable and which are flagged as semantic conflicts for review. Tests separating syntax, semantic and source-data errors.
- Change the SLA to 6 hours and redesign: batches offer no latency guarantee within that window, so the guaranteed subset must use synchronous calls and only a non-urgent remainder can stay in batches. Tests choosing the processing mode from the latency constraint.
- Add a second extraction pass in a separate session that verifies the first pass against the document and compare its flagged items with the confidence routing. Tests the independent-review idea against self-reported confidence.
