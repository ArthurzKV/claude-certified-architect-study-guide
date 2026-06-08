# Domain 4 — Prompt Engineering & Structured Output (20%)

This domain tests whether you can turn a fuzzy task into a reliable, machine-consumable result. The exam frames it through **S6 Structured Data Extraction** (JSON schemas, `tool_use`, validation-retry) and **S2 Code Generation with Claude Code** (clear instructions, examples, chain-of-thought for hard logic). The recurring theme: prefer the most *deterministic* mechanism that still solves the problem, and never trust prose where a schema or a stop condition will do.

Primary sources: Anthropic prompt engineering overview (https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview), tool use / JSON output (https://docs.anthropic.com/en/docs/build-with-claude/tool-use/overview), structured outputs (https://docs.anthropic.com/en/docs/build-with-claude/structured-outputs), and extended thinking (https://docs.anthropic.com/en/docs/build-with-claude/extended-thinking).

## Core Prompt-Engineering Techniques (the documented ladder)

Anthropic documents these techniques roughly in order of impact. The exam expects you to know the order and reach for the *cheapest effective* rung first rather than jumping to fine-tuning-style thinking.

### Be Clear and Direct

Treat Claude like a brilliant new hire with no shared context: spell out the goal, the audience, the format, what to do with edge cases, and what NOT to do. Vague instructions are the #1 cause of "wrong but confident" output. Give sequential steps as numbered lists. State success criteria explicitly so the model can self-check.

**Exam signal:** A distractor will "fix" a flaky prompt by adding more emphatic language ("VERY IMPORTANT!!!"). The correct answer adds *specificity and structure* (explicit format, examples, edge-case rules) — not louder pleading.

### Multishot / Few-Shot Examples

3–5 diverse, relevant examples are the single most reliable way to lock in *format and edge-case handling*. Examples teach the shape of the answer far better than describing it. Wrap each example in tags (e.g. `<example>`) so the model can distinguish them from the live input, and make sure examples cover the tricky cases (nulls, ambiguity, multiple matches), not just the happy path.

**Exam signal:** For "extraction produces inconsistent field formatting / inconsistent casing / inconsistent date formats," the best answer is **few-shot examples that demonstrate the exact target format**, often combined with a schema — not a longer English description of the format.

### Chain-of-Thought (CoT)

For multi-step reasoning, math, complex classification, or anything where the model rushes to a wrong answer, let it think before answering. Structure the thinking (e.g. "First, ... Second, ...") and, when you need clean output, isolate reasoning inside `<thinking>` tags so you can strip it. CoT trades latency/tokens for accuracy; don't apply it to trivial lookups.

**Trap:** Mixing reasoning into the same field as the final structured answer. Keep scratch reasoning in a separate `<thinking>` block (or use extended thinking) so the JSON stays clean and parseable.

### XML Tags for Structure

Claude is specifically tuned to respect XML-style tags. Use them to separate instructions from data from examples (`<document>`, `<instructions>`, `<examples>`, `<formatting>`). This reduces the chance the model treats your data as instructions (a mild prompt-injection mitigation) and makes outputs easier to post-process. Be consistent with tag names and reference them by name in the prompt.

### System / Role Prompts

Put the persona, domain, and standing constraints in the `system` parameter; put the task and data in the user turn. A sharp role ("You are a senior tax analyst extracting fields from W-2 forms") measurably improves accuracy and tone consistency. The system prompt is also the right home for **prompt-cacheable** stable context (see D5) — long instructions, schemas, and examples that don't change between requests.

### Prefill the Assistant Turn

You can seed the start of Claude's response by providing an initial `assistant` message. Prefilling `{` forces the model to begin a JSON object immediately and skip preamble like "Sure, here's the JSON:". Prefilling can also lock voice/format. Note: prefill content is not allowed to end with trailing whitespace — and prefill is a *formatting* lever, not a conformance guarantee. **⚠️ Current-model status:** assistant-message prefill was **removed on current models** (a prefilled final `assistant` turn returns a **400** on Opus 4.6+ / Sonnet 4.5+ and newer). It still works on older Claude 3.x models. For current work, replace prefill with a firm system-prompt instruction plus `output_config.format` (JSON outputs) or `strict: true` tools. See `reference/current-api-errata-2026.md`.

### Prompt Chaining

Decompose a complex job into a sequence of focused prompts, each with one responsibility, passing outputs forward. Chaining beats one mega-prompt for accuracy and debuggability, and it enables a **fresh-context review pass** (extract → validate → critique with a clean prompt). This is the prompt-level analogue of the multi-agent decomposition in D1/S5.

### Long-Context Ordering (the 20K+ rule)

For long inputs, **put the large documents/data near the top of the prompt and the query/instructions at the bottom.** Anthropic documents that placing long-form content at the top can improve response quality by a meaningful margin on long-context tasks. Wrap each document in tags with metadata (source, index), and for grounded Q&A, ask the model to **quote the relevant passages first**, then answer.

**Exam signal:** "We paste a 50-page contract then the question at the very start and accuracy is poor." Fix = reorder so the document is at the top and the question is at the bottom; add per-document tags and quote-first grounding.

### Reducing Hallucinations

Give Claude an explicit "out" — permission to say "I don't know" or return `null` when the answer isn't present. For grounded tasks, require **citations / quotes from the source** before conclusions. Lower temperature for factual extraction. Use CoT to verify before committing. Constrain the output with a schema so the model can't invent extra fields.

**Trap:** A prompt that demands a value for every field with no null/"not found" path *forces* fabrication. Schemas should allow `null` (or a sentinel enum value) for genuinely-absent data.

## Structured Output: Three Approaches and How to Choose

This is the highest-yield cluster for S6. There are three documented ways to get machine-readable output, with increasing strength of the guarantee.

### Approach 1 — `tool_use` with a JSON `input_schema`

Define a tool whose `input_schema` is the JSON Schema you want back, then either let the model call it or set `tool_choice` to force that specific tool. Claude returns a `tool_use` block whose `input` already conforms to the schema's shape. This is the **canonical extraction pattern**: you're not really "calling a tool," you're using the tool interface as a typed output contract.

Use forced `tool_choice` (naming the single extraction tool) so the model can't respond with prose instead of the structured call. The `description` fields on the tool and on each property are real prompt surface — write them carefully.

### Approach 2 — Strict Structured Outputs / Output Schema

Anthropic's structured outputs feature constrains generation so the response is **guaranteed to match the supplied JSON schema** (and, for tools, that tool inputs conform). This is stronger than `tool_use` alone because conformance is enforced during decoding rather than merely encouraged. It is now **generally available (no beta header)** and configured via **`output_config.format`** for the text response (JSON outputs) and **`strict: true`** on a tool for tool-input conformance — two complementary surfaces. When the platform guarantees schema conformance, you can drop most defensive re-prompting for *shape* — though you still validate *semantics* (value ranges, cross-field rules, business validity). Important limits: constrained decoding guarantees only **shape + enums**; it does **not** enforce `minimum`/`maximum`, `minLength`/`maxLength`, or `pattern` (the SDKs strip these into descriptions and post-validate), does **not** support recursive schemas or external `$ref`, requires `additionalProperties: false`, and imposes per-request complexity caps (too many strict tools / optional / union (`anyOf`) params → 400).

**Exam signal:** "We need a hard guarantee the JSON parses and matches the schema every time." Correct answer = use the **strict structured outputs / schema-conformance** feature, not "add more examples" or "ask nicely in the prompt." Examples reduce drift; only schema-constrained decoding *guarantees* shape.

### Approach 3 — Prefill `{` + Stop Sequences

The lightweight (legacy) approach: prefill the assistant turn with `{` to force JSON, and optionally set a `stop_sequence` to halt after the object. Cheap, but it gives **no conformance guarantee** — the model can still emit malformed or extra-field JSON. **⚠️ Removed on current models** (400 on Opus 4.6+ / Sonnet 4.5+); only Claude 3.x still supports it. On current models the lightweight replacement is a firm **system-prompt instruction** ("output only the JSON object") — but prefer Approach 2 (`output_config.format` / `strict: true`) for anything that matters. Always pair any approach with a parse-and-validate step.

### Choosing Among Them

- Need a **hard shape guarantee** → strict structured outputs / schema-constrained decoding.
- Need typed extraction with good ergonomics and broad support → `tool_use` with `input_schema` + forced `tool_choice`.
- Simple, low-stakes formatting → firm system-prompt instruction (or prefill `{` + stop sequence on legacy Claude 3.x only) + validate.
- In all cases for production extraction → **wrap in a validation-and-retry loop.**

## The Validation-and-Retry Loop (S6 core mechanic)

This is the single most testable pattern in S6. The reliable extraction pipeline is:

1. **Request** structured output (tool_use / schema).
2. **Parse** the JSON.
3. **Validate** against the schema AND business rules (required fields present, enums in range, cross-field constraints, plausible values).
4. **On success** → accept.
5. **On failure** → feed the *specific* validation error back to the model in a follow-up turn ("Your output failed validation: `total` (450) does not equal sum of line items (470). Return corrected JSON.") and request a corrective retry.
6. **Bound** the retries with a small cap (e.g. 2–3) as a backstop; on exhaustion, route to human review or a dead-letter queue — do not silently accept.

The corrective signal must be **specific and machine-derived**. Echoing the exact failing field and the exact rule that broke lets the model fix precisely; a generic "that was wrong, try again" wastes a turn.

**Top trap — silently accepting malformed JSON.** Returning empty/partial results as if successful (anti-pattern #7) or swallowing a parse error hides data-quality failures downstream. **Never `try/except: pass`.** A failed extraction must surface, retry, or escalate.

**Top trap — generic error messages.** Feeding back "validation failed" instead of the concrete diagnostic (which field, which rule, the offending value) starves the model of the context it needs to self-correct (anti-pattern #6). Give the model the diagnostic, not a redaction.

**Exam signal:** When a scenario shows extraction "mostly works but occasionally returns broken JSON," the correct architecture is **validate → feed specific error back → corrected retry → bounded cap → escalate**, NOT "increase temperature," "add a bigger model," or "filter out the bad rows."

## Schema Design for Extraction

Good schemas do half the prompt-engineering work, because every `description` is an instruction and every constraint is a guardrail.

- **Required vs optional:** Mark truly-mandatory fields `required`; for fields that may legitimately be absent, allow `null` (or a sentinel) so the model doesn't fabricate. Forcing a value where none exists *causes* hallucination.
- **Enums for closed sets:** Constrain categorical fields (`status`, `priority`, `document_type`) to an `enum`. This eliminates a whole class of formatting drift and makes downstream code total.
- **Descriptions as prompts:** Put extraction rules in each property's `description` ("`invoice_date` in ISO 8601; use the issue date, not the due date"). The model reads these.
- **Types and formats:** Use the narrowest type (`integer`, `number`, `boolean`), and document formats (ISO dates, currency as minor units) explicitly. Add `minimum`/`maximum`/`pattern` where the platform honors them.
- **Avoid over-nesting:** Flatter schemas extract more reliably and are easier to validate. Split genuinely separate concerns into separate tools/passes rather than one giant object.

**Exam signal:** "Free-text `status` field comes back as 'Open', 'OPEN', 'in progress', 'pending-open'..." → the fix is an **`enum`** (plus an example each), not post-hoc string normalization in application code.

## Few-Shot for Consistent Formatting

When the failure mode is *format inconsistency* rather than *wrong content*, few-shot examples are the highest-leverage fix. Show 3–5 input→output pairs that nail the exact casing, date format, null handling, and ordering you want. Pair examples with the schema so shape and style are both pinned. Place examples in the system prompt when stable so they're prompt-cached.

## Extended Thinking for Hard Reasoning

For genuinely hard reasoning (complex multi-constraint extraction, tricky classification, math, multi-step code logic in S2), enable **extended thinking** to give the model a dedicated reasoning budget before the final answer. Key facts to know:

- On current models use **adaptive thinking** (`thinking: {type: "adaptive"}`) and control depth with the **`effort`** parameter (placed inside `output_config`), which calibrates reasoning to query complexity. The older fixed **`budget_tokens`** extended-thinking control is legacy (still functional on some 4.6-class models); for a hard cost ceiling lower `effort` or cap `max_tokens`.
- Extended thinking is for *depth*, not formatting — pair it with a schema/tool for the final structured answer; keep reasoning out of the answer field.
- It costs latency and tokens; reserve it for the hard slices, not every request. For simple high-volume extraction, plain prompting + schema is cheaper and sufficient.
- There are hard interaction constraints: thinking is **incompatible with forced tool use** (`tool_choice` `any`/`tool` error — use `auto`/`none`), and **prefill is unsupported on current models**. Let the model reason, then emit the tool call. When thinking is combined with tool use across turns, **thinking blocks (with their signatures) must be passed back unmodified** in subsequent requests.

**Exam signal:** A scenario with a *hard* per-item reasoning step (resolving conflicting clauses, multi-hop calculation) and accuracy problems → **extended thinking + schema-constrained final output**, not "add more few-shot examples" (examples fix format, thinking fixes reasoning).

## Mapping to the Scenarios

- **S6 Structured Data Extraction:** schema design (enums, required, null), `tool_use`/strict outputs for shape, the validation-retry loop with specific error feedback, per-document-type evaluation (don't let aggregate accuracy hide a failing document class — anti-pattern #10).
- **S2 Code Generation with Claude Code:** clear-and-direct instructions, few-shot for code style, CoT/extended thinking for hard algorithmic logic, XML tags to separate spec from code, and structured outputs when the pipeline consumes the result programmatically (ties to S5 CI/CD).

## Top Traps (memorize)

1. Louder/emphatic prose instead of specificity + examples + schema.
2. Allowing fabrication by requiring a value with no `null`/"not found" path.
3. Prefill `{` treated as a *guarantee* — it isn't; it still needs validation.
4. Silently accepting malformed JSON / `try: ... except: pass` (anti-pattern #7).
5. Generic "validation failed" feedback instead of the specific failing field+rule (anti-pattern #6).
6. Iteration cap as the *only* safety on retries with no escalation path (cap is a backstop — anti-pattern #2).
7. Reasoning mixed into the structured answer field instead of a separate `<thinking>`/thinking block.
8. Free-text where an `enum` belongs; normalizing in app code instead of constraining the schema.
9. Long document placed *after* the question in long-context prompts (degrades quality — put data on top).
10. Aggregate accuracy masking per-document-type failure; few-shot used to fix a *reasoning* problem that needs extended thinking.
