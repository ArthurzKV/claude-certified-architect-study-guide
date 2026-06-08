# Reliable Structured and JSON Output with Claude

This reference covers how to get Claude to emit structured data you can parse with confidence: tool-use schemas, strict structured outputs, prompt-and-prefill techniques, the validate-and-retry loop, schema design, and handling streamed or partial JSON. It maps directly to **Domain 4 (Prompt Engineering & Structured Output, 20%)** and the **S6 Structured Data Extraction** scenario, with strong overlap into **S5 (CI/CD)** where Claude's output feeds downstream automation.

The core exam idea: free-text "please return JSON" is the *weakest* contract. The reliability ladder climbs from prompt-only, to tool-use schemas, to strict structured outputs that guarantee schema-valid JSON. Pick the strongest mechanism the API surface supports, then *still* validate, because a generated value can be schema-valid yet semantically wrong.

Official sources: Anthropic docs on tool use and structured outputs at https://platform.claude.com/docs (and https://docs.anthropic.com/en/docs/build-with-claude/tool-use), plus the Agent SDK docs at https://docs.anthropic.com/en/docs/claude-code.

## Approach A: Tool Use with a JSON input_schema

The most broadly supported way to force structured output is to define a tool whose `input_schema` is the exact shape you want, then let Claude "call" that tool. You are not necessarily executing anything — the tool is a *typed envelope*. Claude returns a `tool_use` content block whose `input` field is a JSON object conforming to your schema.

The `input_schema` is a JSON Schema object (`type`, `properties`, `required`, `enum`, `description`, etc.). To make extraction deterministic, define a single tool (e.g. `record_extraction`) and steer Claude toward it with the `tool_choice` parameter. Setting `tool_choice` to force that specific tool means Claude must emit a `tool_use` for it rather than replying in prose — this is the standard pattern for "extract into this shape, no chit-chat."

Why this beats prompt-only JSON: the model is constrained by the tool-calling machinery and the schema travels with the request, so field names and the enum-vs-free-text distinction are explicit. You read the structure directly from `tool_use.input` instead of regex-scraping a text block.

**Exam signal (S6/D4):** A question describes an extraction pipeline that string-parses Claude's prose reply and breaks on phrasing changes. The correct fix is to define a tool with an `input_schema` and force it via `tool_choice` — not to add more few-shot examples or more pleading in the prompt. "Use a tool schema as the output contract" is the canonical right answer.

## Approach B: Strict Structured Outputs (Guaranteed-Valid JSON)

Where the API and Agent SDK support it, **strict structured outputs** go a step further than tool use: they constrain decoding so the returned JSON is *guaranteed* to validate against your schema. Instead of hoping the model produces well-formed JSON, generation is restricted to tokens that keep the output schema-conformant. This is the strongest contract available.

Conceptually, you attach an output JSON schema to the request and opt into strict mode. The result is structurally valid by construction: required keys present, no trailing prose, types matching the schema. This eliminates the entire class of "the model added a friendly sentence before the JSON" and "it dropped a required field" failures.

Two caveats the exam likes to probe. First, **availability**: strict structured outputs are a specific capability — confirm the model and SDK surface you are targeting supports it before designing around it; if it is unavailable, fall back to Approach A (tool schema). Second, **structural validity is not semantic correctness**: strict mode guarantees the shape, not that an extracted date is the *right* date or that an enum choice is the *correct* category. You still validate semantics and still run an eval. Do not state an exact "supported since version X" number unless you have it in front of you — describe it as "where supported" and verify against current docs at https://platform.claude.com/docs.

**Exam signal (D4/S5):** When the scenario needs downstream code to consume Claude's output with zero parsing tolerance (a CI gate, a batch job writing to a typed store), the best answer is strict structured outputs / an output schema *where supported*, with tool-use schema as the fallback — never "parse the text and hope."

## Approach C: Prompt and Prefill Techniques (Fallback)

When neither tool use nor strict outputs fit, you can raise reliability with prompting. These are weaker than A and B and should be treated as a fallback, but they matter for older surfaces and quick scripts.

**Prefill the assistant turn.** Start the assistant message with an opening token of the structure — e.g. prefill with `{` for JSON or with the opening tag for XML. Prefilling suppresses preamble ("Sure, here's the JSON...") and commits the model to the format immediately. Note that if you prefill `{`, the model continues *after* it, so reattach the brace when reconstructing the full object.

**Use stop sequences.** Pair a prefill with a stop sequence (e.g. stop on the closing structure) so generation halts cleanly at the end of the object and you do not get trailing commentary. This bounds the output and makes parsing predictable.

**Ask explicitly and show the shape.** State "Respond with only a JSON object matching this schema" and include a literal example of the desired structure. XML tags can be easier for the model to delimit reliably than free-form JSON in pure-prompt mode; a common pattern is to request the data inside named tags (`<result>...</result>`) and extract between them. Choose XML or JSON, then be consistent.

**Exam signal (D4):** Prefill + stop sequence is the right "no tool support" answer for forcing clean JSON. The trap answer is relying on a polite instruction alone with no prefill, no stop sequence, and no schema — that is the weakest contract and breaks under model or prompt drift.

## The Validate-and-Retry Loop

Regardless of approach, treat Claude's output as *untrusted until validated*. Parse it, then validate against your schema (e.g. with a JSON Schema validator or a Pydantic/Zod model). On failure, do not crash and do not silently accept — **feed the specific validation error back to the model in a corrective retry.**

The corrective retry is the key technique: include the malformed output (or the offending part) and the exact validator message ("field `total` must be a number, got string `'N/A'`") and ask Claude to fix only that. A targeted error message lets the model self-correct in one pass far more reliably than a blind re-ask. Cap retries with a small bounded number (a safety backstop), and on terminal failure surface a real error with diagnostic context — never a fake success.

**Silently accepting malformed output is an anti-pattern**, and so is its cousin, *silently suppressing the error and returning an empty result as if it succeeded* (anti-patterns #6 and #7 in the exam's canonical list). Both hide failures from the system and from any human in the loop. A generic "extraction failed" with no detail is also wrong, because it strips the diagnostic context the model and the operator need. The right behavior: validate, return the specific error, retry with that error, and fail loudly if it cannot be repaired.

**Exam signal (S6/D4):** The correct extraction design always has a *validation gate after the model*, and a retry that passes the **validation error** back in. Distractors: "trust the model since the schema is strict" (shape ≠ semantics), "retry with the same prompt" (no new information), and "catch the exception and return `{}`" (silent suppression).

## Designing Good JSON Schemas

Schema quality directly drives output quality. The schema is part of the prompt, so write it to be unambiguous.

**Mark required fields explicitly** with `required` so the model cannot quietly omit them. **Use `enum` for closed sets** (status, category, severity) so the model picks from a fixed vocabulary instead of inventing labels — this is one of the highest-leverage reliability moves and makes downstream switch logic safe. **Write a `description` on every field**; descriptions act as inline instructions ("ISO 8601 date", "two-letter country code") and remove guesswork. **Avoid overly deep nesting**: flatten where you reasonably can. Deep, sprawling objects raise error rates and make partial-output recovery harder; prefer a few well-named flat fields or a shallow list of typed records.

Add a typed "unknown / not present" path rather than forcing hallucination. For optional data, either make the field non-required or include an explicit sentinel (e.g. a nullable type), so the model can say "absent" honestly instead of fabricating a value to satisfy `required`.

**Exam signal (D4/S6):** Recognize `enum` + `description` + `required` as the trio that hardens a schema. A scenario where the model invents inconsistent category strings is solved by an `enum`, not by post-hoc string cleanup.

## Handling Partial and Streamed JSON

When you stream responses, JSON arrives incrementally and is *invalid until the final token*. Do not run a strict parser on each chunk and treat exceptions as failures — that is a self-inflicted bug. For tool-use / structured output, accumulate the streamed `input_json` deltas and parse once the block is complete, or use a tolerant partial-JSON parser if you genuinely need to render progressively.

Only validate against your schema after the structure is complete. If a stream is cut off mid-object, that is a real failure: detect it (truncated, missing closing braces, `stop_reason` indicating you hit a length limit) and route it into the retry loop — do not pad or guess the missing fields.

**Exam signal (D5):** Truncated structured output is a reliability concern. The right move is to detect incompleteness via the API signal and retry/repair, not to "best-effort parse whatever arrived." This ties to checking `stop_reason` rather than inferring completion from the text.

## Idempotent Extraction

For pipelines (especially batch and CI in **S5/S6**), make extraction **idempotent**: the same input should yield the same logical record, and re-running must not create duplicates or partial writes. Derive a stable key from the source document (content hash or business ID) and upsert on it, so a retried or replayed extraction overwrites cleanly rather than appending a second row. Keep request and persistence separate: validate first, then commit; never write half-validated output that a retry will duplicate.

For per-item retries inside a batch, scope the retry to the failed item using its stable key so one bad document does not force reprocessing the whole batch. Pair this with **per-slice evaluation**: track accuracy per document type / category, not just an aggregate, because a high overall number can hide a category that fails most of the time (anti-pattern #10).

**Exam signal (S5/S6/D5):** Idempotent, keyed upserts plus per-slice metrics are the "production-grade extraction" markers. The trap is an aggregate accuracy score that masks a failing slice, or a retry that re-inserts a duplicate record because writes were not keyed.

## Quick Decision Guide

Use **strict structured outputs** when the surface supports them and downstream code needs guaranteed-valid JSON. Use a **tool with `input_schema` + forced `tool_choice`** as the strong, broadly-available default. Use **prefill + stop sequence + explicit schema-in-prompt** only when neither is available. In every case: **validate against your schema, retry by feeding the validation error back, and fail loudly with diagnostic context** — silently accepting malformed output, or swallowing the error and returning empty/`{}`, is always the wrong answer on the exam.
