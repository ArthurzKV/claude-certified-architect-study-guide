# Scenario 6: Structured Data Extraction — Architectural Deep-Dive

This is the canonical CCA-F scenario for **turning unstructured documents into validated, machine-usable records**. It exercises Domain 4 (Prompt Engineering & Structured Output) heavily, with strong cross-over into D5 (Context Management & Reliability). The system under test ingests messy inputs — invoices, contracts, resumes, support emails, scanned forms, PDFs — and must emit **structured records that conform to a known schema, every time**. The exam uses S6 to test whether you constrain the *shape* of output deterministically, **validate** it, and **retry with corrective feedback** instead of trusting free-text parsing or accepting whatever the model returns. Most distractors are variations of "just ask for JSON and parse it" or "regex the text."

## Business Problem: Unstructured Documents Into Trustworthy Records

The business has a pile of human-written, inconsistently-formatted documents and needs reliable structured data out of them: an invoice becomes `{vendor, invoice_number, line_items[], total, currency, due_date}`; a resume becomes `{name, email, years_experience, skills[]}`; a contract becomes `{parties[], effective_date, term_months, governing_law}`. The records feed downstream systems — a database, an ERP, an analytics pipeline — that **reject malformed input**. So "looks roughly right" is a failure. The architecture question is never "can Claude read the document" — it is "**what guarantees the output matches the schema, and what happens when it doesn't?**"

Two failure modes the business fears: (1) **silent shape drift** — the model returns valid-looking JSON with a field missing, a wrong type, a hallucinated value, or an enum out of range, and a downstream job crashes or stores garbage; and (2) **silent record loss** — the pipeline quietly drops documents it couldn't parse and reports success, so nobody notices that 8% of invoices never landed. Good architecture makes conformance **enforced and verified**, and makes failures **loud and recoverable**, never swallowed.

## Correct Architecture: Constrain the Shape, Then Validate, Then Retry

The pipeline has three load-bearing parts. **(1) Constrain the output shape** so the model can only produce records in the target structure — use `tool_use` with a precise `input_schema`, or strict **structured outputs**, not "please return JSON" in prose. **(2) Validate** the returned object against the full schema (and any business rules a JSON Schema can't express) in deterministic code. **(3) On validation failure, retry with the specific error fed back** to the model so it can correct, with a bounded retry count and a clean route for records that still fail. See platform.claude.com/docs and docs.anthropic.com for tool use, structured outputs, and the Messages API.

**Constrain with `tool_use` + `input_schema`.** Define a single extraction tool (e.g. `record_invoice`) whose `input_schema` is the JSON Schema for your record. When you force that tool (via `tool_choice`), the model emits a `tool_use` block whose `input` is constrained to the schema's shape, so you get a structured object directly instead of scraping text. This is the standard, well-supported pattern for extraction. Where the platform offers **strict structured outputs**, that guarantees the response conforms to the schema (correct fields, types, required-ness) — prefer it for hard shape guarantees. Either way the principle is the same: the *shape is enforced by the API contract*, not requested politely.

**Exam signal:** If a question asks how to get the model to return data "in a specific shape," the right answer is **`tool_use` with a precise `input_schema`** (or strict structured outputs) — never "ask for JSON in the prompt and `json.loads` it," and never "regex the model's prose."

## Schema Design: Required Fields, Enums, Types, Descriptions, Sensible Nesting

The schema is the contract, and most extraction quality comes from a good one. Apply these design rules:

- **Mark required fields `required`.** This is what makes a missing value a *validation failure* you can act on, instead of a silent `undefined` downstream. Only mark a field required if the document is guaranteed to contain it.
- **Use `enum` for closed value sets.** `currency: {enum: [USD, EUR, GBP, ...]}`, `status: {enum: [paid, unpaid, overdue]}`. Enums stop the model inventing near-miss values ("Dollars", "US$") and make validation trivial.
- **Set precise `type`s and `format`s.** `total: number` not string; `due_date: string, format: date`; `line_items: array of objects`. Wrong types are the most common downstream crash.
- **Write a `description` for every field.** Field descriptions are read by the model and are the cheapest accuracy lever you have: `"invoice_number": "The vendor's invoice ID, usually top-right, e.g. INV-2025-0042. Not the PO number."` disambiguates the two IDs people confuse.
- **Nest deliberately, don't over-nest.** Model genuine structure (`line_items[]` of `{description, qty, unit_price}`) but keep the tree shallow; deep, speculative nesting raises error rate and is harder to validate.
- **Constrain where you can.** `minItems`, `minimum/maximum`, `pattern` for known ID formats. Each constraint converts a class of bad output into a catchable validation error.

**Exam signal:** "How do you stop the model returning the wrong currency string / a string where a number belongs?" → **enums and precise types in the schema**, enforced by validation — not a sentence in the prompt asking nicely.

## The Validation-Retry Loop: Validate, Return the Specific Error, Retry, Never Silently Accept

This loop is the single most-tested mechanic in S6. After the model returns a record, **validate it against the schema** (JSON Schema validation) *plus* any business rules schema can't express (e.g. `sum(line_items.amount) == total`, `due_date >= invoice_date`). Three outcomes:

- **Valid** → accept the record, emit it downstream.
- **Invalid** → **do not accept it, and do not silently drop it.** Build a precise, machine-actionable error message ("`total` is required but missing"; "`currency` value 'Dollars' not in enum [USD,EUR,...]"; "line-item amounts sum to 240.00 but `total` is 260.00") and **send it back to the model as a corrective `tool_result` / follow-up turn**, asking it to fix exactly those fields. The model retries with the error in context. This is the corrective retry.
- **Still invalid after N retries** → route the record to a **needs-review / dead-letter queue** with the document, the last output, and the accumulated errors attached. It is counted as a failure, surfaced, and reprocessable — never reported as success.

Two non-negotiables the exam hammers: **(a) the retry error must be specific** — returning a generic "invalid, try again" strips the model of the information it needs to fix the right field (anti-pattern #6); **(b) never silently accept or silently empty** — returning `{}`, dropping the record, or marking a failed parse as success is anti-pattern #7 and the most dangerous bug in an extraction pipeline, because the data loss is invisible. Bound the retries (a small N) so a pathologically bad document can't loop forever, but the **bound is a backstop, not the design** — the control is validate-and-correct.

**Exam signal:** "The extracted record fails schema validation — what should happen?" → **return the specific validation error to the model and retry the correction**, then route to needs-review on exhaustion. Wrong answers: accept it anyway, return an empty record as success, silently drop it, or retry with a vague "that was wrong."

## Missing and Ambiguous Fields: Explicit Nulls and Needs-Review From Verifiable Checks

Documents legitimately lack fields, and conflating "absent" with "wrong" corrupts data. Design for it:

- **Allow explicit `null` for genuinely-optional fields** (e.g. `purchase_order_number`) — make the schema permit `null` and instruct the model to emit `null` when the value is truly not present, rather than guessing or hallucinating a plausible value. An explicit `null` is honest data; a fabricated PO number is corruption.
- **Carry a `needs_review` flag, set from VERIFIABLE checks** — not from the model's vibe. Flag a record for human review when a *deterministic condition* holds: a required field came back null, an arithmetic check failed (line items don't sum to total), a date is implausible, or a value barely passed a `pattern`. The flag must be something you could write an assertion about and log — exactly the discipline from S1's escalation logic.
- **Do NOT use a model self-reported confidence score (e.g. "confidence: 0.6") to drive review routing.** That number is uncalibrated and the model can be confidently wrong (anti-pattern #4). If you want a "confidence," derive it from the verifiable checks (did totals reconcile? did all required fields populate? did enums match?), not from asking the model to rate itself.

**Exam signal:** "How should the system handle a field that isn't in the document?" → **emit an explicit `null` and, if the field is important, set a `needs_review` flag from a deterministic check** — not invent a value, and not route on a self-rated confidence number.

## Few-Shot Examples and Prefill + Stop Sequences

For **tricky or ambiguous formats**, add a small number of **few-shot examples** showing input → exact target record, including the hard cases (a missing field rendered as `null`, an unusual date format normalized, two confusable IDs disambiguated). A few well-chosen examples teach the desired normalization far more reliably than longer instructions, and they pin down edge behavior the schema alone can't express.

**Prefill + stop sequences** is a legitimate *lightweight* method when you are not using tool_use/structured outputs (e.g. a quick extraction or a model/path without forced tools). Prefill the assistant turn with the opening of the structure (e.g. `{` or `{"invoice_number":`) to force the model straight into the format and skip preamble, and set a **stop sequence** to cut generation cleanly at the end of the object. It is cheaper than a tool call but gives **weaker guarantees** than `input_schema`/strict outputs — you still must validate. Treat prefill+stop as a tactic, tool_use/structured outputs as the default for anything that must conform.

**Exam signal:** Few-shot is the answer for "the model keeps mishandling an unusual/edge format." Prefill+stop is the answer for "a lightweight way to force JSON shape without a tool" — but it does **not** replace validation.

## Batch API for Large Document Sets

When the workload is **a large corpus processed offline** (thousands of invoices overnight, a back-catalog of contracts) rather than interactive, use the **Message Batches / Batch API**. It is built for high-throughput, asynchronous, non-latency-sensitive jobs: you submit many independent extraction requests, they process in bulk, and you collect results — typically at **lower cost** than the same volume of synchronous calls, and without you managing per-request concurrency and rate limits by hand. Each batch item still carries the same tool_use schema, and **validation-retry still applies per record** — batch changes the *delivery mechanism*, not the conformance discipline. See platform.claude.com/docs for the Batches API. (Note: batch is for throughput/cost on large async sets; for a single interactive document you call the Messages API directly.)

**Exam signal:** "Extract from 50,000 archived documents overnight, cost matters, latency doesn't" → **Batch API**. "Extract from this one document a user just uploaded" → synchronous Messages API call. Batch is not a substitute for validation.

## Per-Document-Type Accuracy Metrics: Evaluate Per Slice, Not in Aggregate

You must measure extraction quality, and **a single aggregate accuracy number is a trap** (anti-pattern #10). A pipeline can report "96% field accuracy" while being catastrophically broken on one document type — if invoices are 95% of volume and extract perfectly but contracts are 5% and extract at 40%, the aggregate hides a total failure on contracts. **Evaluate per slice**: per document type, per field, per vendor/template, per language. Track which *fields* fail most (often dates and totals), which *document types* drag, and what fraction land in needs-review. Build evals from a labeled golden set and compute per-slice pass rates so a regression on one category can't be masked by volume elsewhere. See docs.anthropic.com on evals/test-driven development for agents.

**Exam signal:** "How do you know the extraction is working?" → **per-document-type / per-field accuracy on a labeled eval set**, not one aggregate number. The distractor is always the single global accuracy figure.

## Decision Tree: Extract a Record From a Document

```
Document arrives
        │
        ▼
Large async corpus? ──► YES ──► route via Batch API (same schema, same validation)
        │ NO
        ▼
Define extraction tool with precise input_schema
(required fields, enums, types, descriptions, sensible nesting; null allowed for optional)
        │
        ▼
Force tool_use (or strict structured output); add few-shot for tricky formats
(prefill+stop only if not using tools — still validate)
        │
        ▼
Model returns candidate record
        │
        ▼
VALIDATE: JSON Schema + business rules (sums reconcile, dates sane, enums in range)
        │
        ├─ Invalid ──► build SPECIFIC error ──► return to model as corrective retry
        │                    │
        │                    └─ still invalid after N retries ──► NEEDS-REVIEW / dead-letter
        │                          (record + outputs + errors attached; counted as FAILURE, surfaced)
        │
        └─ Valid
              │
              ▼
        Required field null OR check failed (totals, date, pattern)?
              ├─ YES ──► set needs_review = true (from VERIFIABLE check, not self-confidence)
              └─ NO  ──► emit record downstream
        │
        ▼
Score quality PER document-type / PER field on labeled eval set
(never a single aggregate accuracy number)
```

## Likely WRONG Answers (Distractors to Eliminate)

- **Regex / string-parsing the model's free-text output**, or "ask for JSON in the prompt and `json.loads()` it." Use **`tool_use` + `input_schema`** or strict structured outputs; the shape must be enforced, not parsed.
- **No validation step** — accept whatever the model returns. You must validate against the full schema *and* business rules.
- **On a parse/validation failure, return an empty record or drop the document and report success.** Silent suppression / empty-as-success is anti-pattern #7; route to needs-review and surface it.
- **Retry with a vague "that was invalid, try again"** instead of the specific failing-field error. Generic errors hide diagnostic context (anti-pattern #6).
- **Hallucinate a value for a missing field** instead of emitting an explicit `null`. Fabricated data is worse than absent data.
- **Route records to human review based on the model's self-reported confidence score.** Use verifiable checks, not a self-rating (anti-pattern #4).
- **Report one aggregate accuracy number.** Measure per document-type and per field (anti-pattern #10).
- **Enforce field rules (types, allowed values) by instructions in the prompt only.** Encode them in the **schema (enums/types) and validation**, deterministically (echoes anti-pattern #3).
- **Use an arbitrary retry cap as the primary correctness mechanism.** The control is validate-and-correct; the cap is only a backstop (anti-pattern #2).

## "If the question asks X, the answer is Y" Mappings

- **If the question asks** "how do you make the model return data in a specific structure?" → **`tool_use` with a precise `input_schema`** (or strict structured outputs) — never prompt-and-parse, never regex.
- **If the question asks** "the extracted record fails schema validation — what should the pipeline do?" → **return the specific validation error to the model and retry the correction**, bounded; route to needs-review/dead-letter on exhaustion. Never accept-anyway, empty-as-success, or silent drop.
- **If the question asks** "a field isn't present in the document — how should it be represented?" → **explicit `null`** (schema permits it), plus a **`needs_review` flag set from a verifiable check** if the field matters — not a hallucinated value.
- **If the question asks** "how do you stop the model emitting an invalid currency / a string where a number is required?" → **enums and precise types in the schema**, enforced by validation.
- **If the question asks** "the model keeps mishandling an unusual date/ID format — what helps most?" → **few-shot examples** of that exact edge case mapping to the target record.
- **If the question asks** "a lightweight way to force JSON shape without a tool call?" → **prefill the assistant turn + a stop sequence** — but you **still validate**; it's weaker than `input_schema`.
- **If the question asks** "extract from a huge archive overnight where cost matters and latency doesn't?" → **Batch API**, with the same schema and the same validation-retry per record.
- **If the question asks** "how do you know extraction quality is acceptable?" → **per-document-type and per-field accuracy on a labeled eval set** — not a single aggregate accuracy number.
- **If the question asks** "should a needs-review decision use the model's confidence score?" → **no — derive review routing from verifiable checks** (required nulls, failed reconciliations, out-of-range values), not a self-rating.
