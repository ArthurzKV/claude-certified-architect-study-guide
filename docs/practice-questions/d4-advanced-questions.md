# Domain D4 — Advanced & Edge-Case Questions

These advanced, edge-case practice questions extend **Domain 4 (Prompt Engineering & Structured Output, 20%)** of the Claude Certified Architect – Foundations (CCA-F) exam. They target high-yield gaps under-represented in the base bank: the **current** strict structured outputs surface (`output_config` JSON outputs vs `strict: true` tool use), the documented JSON-Schema feature limits under constrained decoding, complexity limits, SDK schema-transformation behavior, the shift from prefill and `budget_tokens` extended thinking toward adaptive thinking and `effort`, streaming partial-JSON handling, per-slice evaluation, idempotent persistence, and the reasoning-vs-format diagnostic split.

This is original study material grounded in publicly documented Anthropic behavior (prompt engineering, tool use, and structured outputs at https://platform.claude.com/docs and https://docs.anthropic.com/en/docs/build-with-claude). It is NOT a dump of live or proctored exam questions. Where an exact API value is uncertain, the behavior is described conceptually rather than invented.

Each question is tagged with difficulty and anchoring scenario on a `Tags:` line under the explanation.

## Section A — Strict Structured Outputs: Surface and Mechanism

### Q1. In S6, a team wants a hard guarantee that Claude's *final text response* parses as JSON matching a schema, while a separate team wants a guarantee that *tool inputs* conform. Which mechanism maps to each need on the current API?

A) Both needs are served only by prefilling `{` and validating
B) JSON outputs via `output_config.format` (type `json_schema`) constrain the text response; `strict: true` on a tool constrains that tool's input parameters — they are complementary and can be used independently or together
C) Both are served by setting `tool_choice` to `auto`
D) Only the system prompt can guarantee either; schemas merely suggest shape
Answer: B
Explanation: Strict structured outputs ship as two complementary features: `output_config.format` (json_schema) constrains the assistant's text response, and `strict: true` on a tool grammar-constrains that tool's inputs. You can apply them independently or together. Prefill gives no guarantee, `tool_choice: auto` is about whether to call a tool (not conformance), and prose instructions cannot guarantee shape.
Tags: D4 | S6 | Advanced | output-config-vs-strict-tool

### Q2. Which statement about how strict structured outputs enforce conformance is correct?

A) The API validates the JSON after generation and silently drops invalid fields
B) Conformance is enforced during decoding via grammar-constrained sampling, so the response is valid by construction rather than validated after the fact
C) The model is merely prompted with the schema and asked to comply
D) Conformance is guaranteed only when temperature is set to 0
Answer: B
Explanation: Strict structured outputs use grammar-constrained (constrained) decoding so the tokens that violate the schema are never sampled — validity by construction, not post-hoc cleanup. It is not a prompt-only nudge, does not silently drop fields, and does not depend on temperature.
Tags: D4 | S6 | Advanced | constrained-decoding-mechanism

### Q3. A schema for strict structured outputs includes `minimum: 0`, `maximum: 100`, and `pattern` on several fields, plus `minLength` on a string. What should an architect expect on the current Anthropic surface?

A) All of these constraints are enforced by the constrained decoder at generation time
B) Numerical (`minimum`/`maximum`), string-length (`minLength`/`maxLength`), and regex `pattern` constraints are NOT enforced by strict structured outputs; the official SDKs strip unsupported constraints, move them into field descriptions, and validate the response against the original schema afterward
C) The request is rejected outright because only `enum` is allowed
D) The constraints silently disable strict mode for the whole request
Answer: B
Explanation: Documented strict-mode support covers basic types, `enum`, `const`, `anyOf`/`allOf`, `$ref`, `required`, and `additionalProperties: false`, but NOT numeric ranges, string-length, or regex `pattern`. The Python/TypeScript/Ruby/PHP SDKs transform unsupported schemas by removing those constraints, folding them into descriptions, and validating responses against your original schema. So range/pattern checks become *post-generation validation*, not decode-time guarantees.
Tags: D4 | S6 | Advanced | strict-unsupported-constraints

### Q4. Why does it matter that `minimum`/`maximum`/`pattern` are enforced by post-generation SDK validation rather than the strict decoder?

A) It does not matter; the effect is identical
B) Because a value can be grammar-valid yet out of range or mis-formatted, so range/regex violations still require a validation step and a corrective retry — they are not prevented by constrained decoding
C) Because it forces every request into batch mode
D) Because it disables enums elsewhere in the schema
Answer: B
Explanation: Constrained decoding guarantees structural shape and enum membership, but range and pattern rules are checked *after* generation by the SDK, so a number can decode in-shape yet violate `minimum`. That means your validation-and-retry loop must still cover ranges/patterns/semantics; do not assume strict mode catches them.
Tags: D4 | S6 | Advanced | decode-vs-postvalidation

### Q5. A complex extraction request defines 22 strict tools, each with many optional and union-typed parameters, and the API returns a 400 error. What is the most likely documented cause?

A) Strict mode forbids more than one tool per request
B) The request exceeds strict-output complexity limits (e.g. the maximum number of strict tools, total optional parameters, or union-typed parameters), so the compiled grammar is too large — flatten/split or reduce optional/union parameters
C) Union types are never allowed in any schema
D) The model ran out of output tokens
Answer: B
Explanation: Strict structured outputs impose explicit complexity ceilings (a cap on strict tools, on total optional parameters, and on union-typed parameters), plus internal limits on compiled grammar size; exceeding them returns a 400. The fix is to reduce optional/union parameters, flatten nesting, or split into multiple requests. It is not a one-tool limit, unions are allowed within limits, and this is a request-validation error, not output truncation.
Tags: D4 | S6 | Advanced | strict-complexity-limits

### Q6. To make strict structured outputs behave predictably, why is `additionalProperties: false` significant?

A) It is ignored by strict mode
B) Strict structured outputs require `additionalProperties` to be `false` (not an arbitrary subschema), which guarantees the object contains exactly the declared properties and no surprise extra keys
C) It enables `minimum`/`maximum` enforcement
D) It switches the response from JSON to XML
Answer: B
Explanation: Under strict outputs, `additionalProperties` must be `false`; this closes the object to exactly the declared keys so downstream code can be total over the field set. It does not enable numeric constraints, is not ignored, and has nothing to do with XML.
Tags: D4 | S6 | Advanced | additionalproperties-false

### Q7. A schema uses a recursive `$ref` to model a tree of nested comments. Under strict structured outputs, what should the architect anticipate?

A) Recursion is fully supported and grammar-constrained
B) Recursive schemas are not supported by strict structured outputs, so a recursive shape must be redesigned (e.g. bounded depth / flattened) or handled with tool_use plus post-validation rather than strict constrained decoding
C) Recursion forces the request into extended thinking
D) Recursion only fails when combined with enums
Answer: B
Explanation: Documented strict-mode limits exclude recursive schemas (and external `$ref`). A self-referential tree must be bounded/flattened, or you fall back to a non-strict tool schema with explicit post-validation. It does not trigger thinking and does not depend on enums.
Tags: D4 | S6 | Advanced | strict-no-recursion

### Q8. Which statement about the strict-outputs *beta header* is correct for the current API?

A) You must always send a `structured-outputs` beta header or requests are rejected
B) Strict structured outputs are generally available and no special beta header is required; you configure them with the standard `output_config` parameter (and `strict: true` on tools)
C) The feature is streaming-only and requires a streaming beta header
D) The feature is deprecated in favor of prefill
Answer: B
Explanation: Strict structured outputs are GA on current Claude models and are configured through the standard `output_config` parameter (plus `strict: true` for tool inputs); the older beta header is no longer required. It is not streaming-only and is not deprecated in favor of prefill — prefill is the weaker, partly-retired mechanism.
Tags: D4 | S6 | Intermediate | strict-ga-no-beta-header

## Section B — Prefill Is No Longer a Live Lever on Current Models

### Q9. An architect's runbook still says "prefill the assistant turn with `{` to force JSON." Targeting current Claude models (4.6 and newer, including Opus 4.x), why is this guidance stale?

A) Prefill still works identically on all models
B) Prefilled responses on the last assistant turn are no longer supported on current models and return a 400 error; the documented replacement for forcing JSON is strict structured outputs (`output_config` / strict tools), and clear prompting in the user turn for the other prefill use cases
C) Prefill now requires a beta header but otherwise works
D) Prefill was replaced by sentiment analysis
Answer: B
Explanation: Starting with Claude 4.6-class models, a prefilled final assistant turn returns a 400 error; the capability that prefill used to provide (force JSON, skip preamble, format steering) is now served by strict structured outputs or clear user-turn prompting. Earlier models still accept prefill, and adding non-final assistant messages mid-conversation is unaffected — but a runbook that relies on last-turn prefill on current models is broken.
Tags: D4 | S6 | Advanced | prefill-deprecated-current-models

### Q10. On a current model, an extraction service still prepends "Sure, here's the JSON:" before the object. The team can no longer prefill `{`. What is the correct fix?

A) Re-enable prefill with a flag
B) Use JSON outputs via `output_config.format` (or a forced/strict tool) so the response is constrained to the object with no preamble, validating semantics afterward
C) Add "do not say anything before the JSON" five times to the system prompt
D) Switch to a streaming request to strip the preamble
Answer: B
Explanation: With last-turn prefill unsupported on current models, the documented replacement for "force the object, skip preamble" is strict structured outputs (JSON outputs or a strict/forced tool), which constrains the surface to the object. Repeated pleading is unreliable, streaming does not remove a preamble token-by-token in a guaranteed way, and re-enabling prefill is not an option on these models.
Tags: D4 | S6 | Advanced | prefill-replacement-json-outputs

### Q11. On a current model, is adding an assistant message that is NOT the final turn (a genuine mid-conversation prior turn) still allowed, or is it rejected like last-turn prefill?

A) Always rejected with a 400 like last-turn prefill
B) Still allowed; the deprecation specifically targets a *prefilled final assistant turn*, not assistant messages elsewhere in the conversation history
C) Only allowed in batch mode
D) Allowed only if it ends with whitespace
Answer: B
Explanation: The restriction is on prefilling the *last* assistant turn to steer the next generation; ordinary assistant messages earlier in the conversation (genuine history) are unaffected. It is not batch-gated, and the old "no trailing whitespace" rule applied to prefill content that is now unsupported on the final turn anyway.
Tags: D4 | S6 | Advanced | midconversation-assistant-allowed

### Q12. A legacy prompt relied on prefill to "resume" a response that was cut off by `max_tokens`. On a current model, what is the recommended continuation strategy?

A) Reattach the prefilled fragment and hope the brace count matches
B) Re-request with the prior partial content placed in the user turn as context (or raise `max_tokens` / use a length-aware retry), since last-turn prefill is unsupported; do not rely on prefilled continuation
C) Ignore the truncation and ship the partial object
D) Lower temperature so it stops needing continuation
Answer: B
Explanation: Because last-turn prefill is unsupported on current models, the documented migration moves prefilled context into the user turn (or you simply raise the token budget and retry with a completeness check). Shipping a truncated object is silent suppression, and temperature does not control truncation.
Tags: D4 | S6 | Advanced | prefill-continuation-migration

## Section C — Adaptive Thinking, Effort, and the budget_tokens Shift

### Q13. A guide says "enable extended thinking with a `budget_tokens` allowance for hard reasoning." For current Claude models, what is the more current recommendation?

A) Always set `budget_tokens` as high as possible
B) Current models use adaptive thinking (`thinking: {type: "adaptive"}`), where the model calibrates how much to think based on the `effort` parameter and query complexity; `budget_tokens`-style extended thinking still functions on some models but is deprecated — prefer adaptive thinking with `effort` (or `max_tokens` as a hard ceiling)
C) Thinking must be enabled on every request for safety
D) Thinking and schemas are mutually exclusive
Answer: B
Explanation: Anthropic documents adaptive thinking as the current approach: the model decides when and how much to reason, driven by `effort` and query complexity, and reports it reliably beats fixed extended thinking in internal evals. The older `budget_tokens` extended-thinking control is deprecated though still functional on some models. Thinking is off by default when omitted and composes fine with schemas.
Tags: D4 | S6 | Advanced | adaptive-thinking-vs-budget-tokens

### Q14. On a current adaptive-thinking model, which two factors most directly govern how much the model reasons before answering?

A) Temperature and `top_p`
B) The `effort` parameter and the model's judgment of query complexity (higher effort and harder queries elicit more thinking; easy queries may get a direct answer)
C) The number of few-shot examples and the system prompt length
D) The retry cap and the stop sequence
Answer: B
Explanation: Adaptive thinking calibrates on `effort` plus assessed query complexity; simple queries can be answered directly with little or no thinking. Sampling parameters, example count, and the retry cap do not govern thinking depth.
Tags: D4 | S6 | Advanced | effort-and-complexity-drive-thinking

### Q15. A team wants a hard ceiling on reasoning cost on a current model without using deprecated `budget_tokens`. What is the documented approach?

A) Set a low temperature
B) Lower the `effort` setting and/or use `max_tokens` as a hard limit alongside adaptive thinking
C) Disable the schema to save tokens
D) Increase the retry cap so it stops sooner
Answer: B
Explanation: To cap reasoning cost on current models, lower `effort` and/or set `max_tokens` as a hard ceiling with adaptive thinking, rather than the deprecated `budget_tokens`. Temperature does not bound cost, dropping the schema loses the contract, and the retry cap is unrelated to per-request thinking spend.
Tags: D4 | S6 | Advanced | effort-maxtokens-cost-ceiling

### Q16. When extended/adaptive thinking is DISABLED on certain current models, the docs note a sensitivity worth knowing. What is it?

A) The model ignores all XML tags
B) With thinking disabled, the model can be unusually sensitive to the word "think" and its variants; using alternatives like "consider," "evaluate," or "reason through" can give better results
C) The schema is ignored unless thinking is on
D) Enums stop working without thinking
Answer: B
Explanation: Anthropic documents that with thinking disabled, certain models (e.g. Opus 4.5) are particularly sensitive to "think" and its variants, and suggests wording like "consider" / "evaluate" / "reason through." XML tags, schemas, and enums all continue to function regardless of thinking state.
Tags: D4 | S2 | Advanced | think-word-sensitivity

### Q17. You want few-shot examples to teach the *reasoning pattern* a thinking-enabled model should follow. What is the documented technique?

A) It is impossible; thinking blocks cannot be demonstrated
B) Include `<thinking>`-style reasoning inside your few-shot examples so the model generalizes that reasoning style to its own thinking blocks
C) Put the reasoning in the schema `description` fields
D) Raise the retry cap so the model reasons longer
Answer: B
Explanation: Multishot examples work with thinking: showing `<thinking>` reasoning inside examples teaches the model the reasoning pattern it should generalize. Schema descriptions steer field extraction, not the reasoning trajectory, and the retry cap is unrelated to reasoning depth.
Tags: D4 | S6 | Advanced | thinking-in-fewshot

## Section D — Reasoning vs Format: The Diagnostic Split (Edge Cases)

### Q18. An S6 model returns perfectly-shaped JSON (passes strict validation) but the extracted `tax_owed` is computed wrong on multi-rate invoices. Examples and a schema are already in place. What is the correct lever?

A) Add more few-shot examples of the output JSON shape
B) Apply reasoning depth (chain-of-thought or adaptive/extended thinking) for the calculation, then emit the result through the schema — shape is fine, the *reasoning* is the failure
C) Tighten the enum on an unrelated field
D) Lower temperature and ship
Answer: B
Explanation: Schema-valid-but-wrong-value is a reasoning failure, not a format failure; the fix is reasoning depth (CoT / thinking) with the schema kept for the clean final answer. More shape examples and unrelated enum tweaks address format, which is already correct; temperature alone does not give room to reason through multi-rate math.
Tags: D4 | S6 | Advanced | reasoning-not-format

### Q19. A classifier returns one of five valid enum labels every time (shape is guaranteed) but mislabels ambiguous edge documents. Which intervention best targets the actual problem?

A) Add `minimum`/`maximum` to the enum
B) Add diverse few-shot examples of the *ambiguous* cases with the correct label and brief reasoning, and/or apply thinking on hard items — the failure is judgment on edge cases, not output shape
C) Switch from a tool schema to prefill
D) Raise the retry cap
Answer: B
Explanation: Guaranteed-valid-but-wrong-label is a judgment problem on edge cases. The leverage is examples that demonstrate the hard/ambiguous cases (plus optional thinking), not numeric constraints (meaningless on enums), not prefill (weaker contract, and unsupported on current models), and not a higher cap.
Tags: D4 | S6 | Advanced | enum-edgecase-judgment

### Q20. Few-shot examples and thinking are sometimes confused. Which pairing is correct?

A) Examples fix multi-hop reasoning; thinking fixes casing/date drift
B) Few-shot examples primarily fix *format/consistency* (casing, dates, null handling, ordering); chain-of-thought/thinking primarily fixes *reasoning* (multi-step math, conflicting-clause resolution, classification judgment)
C) Both fix only format
D) Both fix only reasoning
Answer: B
Explanation: The recurring exam split: examples lock format and edge-case *shape*; thinking adds reasoning *depth*. Mapping them backwards (examples for math, thinking for casing) is the classic trap. They are complementary, each addressing a different failure class.
Tags: D4 | S6 | Intermediate | examples-vs-thinking-split

## Section E — Streaming, Partial JSON, and Completeness Signals

### Q21. A service streams a `strict: true` tool call and runs `json.loads` on every accumulated chunk, treating each exception as a hard failure that triggers a retry. Why is this wrong?

A) Streaming is unsupported for strict tools
B) The `input_json` is incrementally streamed as partial JSON deltas that are invalid until the block completes; you accumulate the deltas and parse/validate once the tool_use block is finished (or use a tolerant partial parser) — mid-stream parse errors are expected, not failures
C) Strict mode disables streaming entirely
D) The parser must run twice per chunk
Answer: B
Explanation: Tool input arrives as partial-JSON deltas that are not valid JSON until complete; strict parsing every chunk will always throw mid-stream and would spuriously trigger retries. Accumulate deltas, then parse/validate at block completion. Streaming is supported for tool use including strict tools.
Tags: D4/D5 | S6 | Advanced | streaming-partial-json-deltas

### Q22. A streamed structured response ends with a length-related `stop_reason` and the JSON object is missing its closing braces. What is the correct handling?

A) Best-effort parse and fill missing fields with defaults
B) Treat it as a truncation/incompleteness signal, route the item into the retry/repair loop (raise the budget or re-request), and never fabricate the missing fields
C) Accept it as a successful partial extraction
D) Lower temperature and accept the partial object
Answer: B
Explanation: A length-limit stop with an unterminated object is a real failure detectable via the API signal; the correct move is retry/repair (e.g. raise `max_tokens`), not padding with defaults (fabrication) or accepting it (silent suppression). Temperature is irrelevant to truncation.
Tags: D4/D5 | S6 | Advanced | truncation-stopreason-repair

### Q23. With strict JSON outputs guaranteeing shape, a developer concludes "we can delete the parse-error branch entirely." What edge case still breaks this assumption?

A) None; strict outputs make parsing infallible end to end
B) Streaming truncation and transport/length limits can still cut off a response mid-object, and unsupported constraints (ranges/patterns) are only post-validated — so completeness checks and semantic validation remain necessary
C) Strict outputs only guarantee the first field
D) Strict mode randomly emits XML
Answer: B
Explanation: Constrained decoding guarantees the *grammar* of what is produced, but a stream can still be truncated by length limits (yielding an incomplete object), and range/pattern/semantic rules are validated after generation. So completeness and semantic checks stay in the pipeline. Strict mode does not emit XML or cover only one field.
Tags: D4/D5 | S6 | Advanced | strict-still-needs-completeness

## Section F — Evaluation, Idempotency, and Per-Slice Reliability

### Q24. An S5/S6 extraction eval reports a single aggregate F1 of 0.95 across all document types, and the team ships. A finance subteam later finds receipts with foreign-currency totals are wrong ~40% of the time. What does the exam want here?

A) Ship anyway; 0.95 clears the bar
B) Break the eval down per document type / per field / per slice so a failing category cannot hide behind a healthy aggregate, then fix the failing slice at the root (anti-pattern #10)
C) Filter foreign-currency receipts out of the dataset and re-report 0.97
D) Raise the retry cap until the aggregate improves
Answer: B
Explanation: Aggregate metrics mask per-slice failures (anti-pattern #10); the fix is per-document-type / per-field evaluation, then a root-cause fix on the failing slice. Filtering out the failing class is the blocking/filtering anti-pattern (hiding, not fixing), and a higher cap does not address the slice.
Tags: D4/D5 | S5/S6 | Advanced | per-slice-eval-foreign-currency

### Q25. An idempotency edge case: the same source document is reprocessed after a transient timeout, and a content-hash upsert key is used. A field that legitimately changed in a corrected re-scan now overwrites the old row. Is this correct behavior?

A) No — upsert is always wrong for extraction
B) Yes — deriving a stable key from the source (content hash or business ID) and upserting is the documented idempotent pattern; a replay overwrites cleanly without creating duplicates, which is the intended outcome
C) No — you should append a new row on every run to preserve history
D) Yes, but only if the key is a random UUID
Answer: B
Explanation: Stable-key upsert is the documented idempotent persistence pattern: replays/retries overwrite the same row instead of duplicating. A random UUID defeats idempotency, and append-on-every-run produces duplicate records. (If you need history, that is a versioning decision layered on top, not a reason to abandon idempotent keys.)
Tags: D4/D5 | S5/S6 | Advanced | idempotent-upsert-edge

### Q26. In a batch of 10,000 documents, document #7,432 fails validation three times and exhausts its per-item retry cap. What is the correct terminal behavior for that item?

A) Fail the entire 10,000-document batch and report failure
B) Route that single item to human review or a dead-letter queue with its diagnostic context, mark it unresolved, and let the other 9,999 succeed — never silently accept it or fake success
C) Return an empty object for it and mark the batch fully successful
D) Loop on it forever until it parses
Answer: B
Explanation: The cap is a backstop; on exhaustion you surface the failure with diagnostics and escalate that *item* (dead-letter / human review) while the rest of the batch proceeds. Failing the whole batch is wasteful, empty-as-success is silent suppression (anti-pattern #7), and an unbounded loop defeats the cap.
Tags: D4/D5 | S5/S6 | Advanced | per-item-deadletter-terminal

### Q27. Which corrective-retry message gives the model the best chance to self-correct a range violation that strict mode did NOT prevent?

A) "Your previous answer was invalid. Try again."
B) "Field `confidence` must be between 0 and 1 (inclusive); you returned 1.4. Return corrected JSON conforming to the schema." — naming the field, the rule, and the offending value
C) "Lower your temperature and retry."
D) Re-send the identical prompt with no diagnostic
Answer: B
Explanation: Range/pattern checks are post-generation, so a specific, machine-derived diagnostic (field + rule + offending value) is exactly what lets the model fix it in one pass (avoiding anti-pattern #6, generic errors). A generic "invalid," a temperature instruction, or an unchanged re-send all starve the model of the needed context.
Tags: D4/D5 | S6 | Advanced | specific-range-retry-message

## Section G — Schema Design Under Constrained Decoding (Edge Cases)

### Q28. Given that strict structured outputs do NOT enforce `minimum`/`maximum`/`pattern`, where should those rules still be expressed, and why?

A) Drop them entirely; unenforced rules are useless
B) Keep them in the schema AND/OR fold them into field `description`s (the SDKs do this automatically when transforming unsupported schemas), and enforce them in post-generation validation — they still guide the model and remain the contract your validator checks
C) Move them to the system prompt only and remove the schema
D) Replace every constrained number with a free-text string
Answer: B
Explanation: Even though the decoder does not enforce ranges/patterns, the SDK transformation moves them into descriptions (which the model reads) and validates responses against your original schema, so they still steer generation and define what your validator enforces. Dropping them loses the contract; free-text reintroduces drift; schema-less system-prompt rules are weaker and unvalidated.
Tags: D4 | S6 | Advanced | unenforced-constraints-still-useful

### Q29. An architect models an optional field as `anyOf: [{type: "string"}, {type: "null"}]` to allow "absent." Under strict outputs with many such optional/union fields, what limit must they watch?

A) None; unions and optionals are unlimited
B) Strict outputs cap the total number of optional parameters and the number of union-typed (`anyOf`) parameters per request; nullable-via-union on many fields can hit those limits and 400 — consider reducing optional/union fields or splitting the request
C) Nullable fields are forbidden entirely
D) Unions silently disable enums
Answer: B
Explanation: Nullable-via-`anyOf` is a legitimate "absent" path, but it counts against the documented caps on optional and union-typed parameters; over-using it across many fields can exceed limits and return a 400. The fix is to reduce optional/union parameters or split the extraction. Nullable fields are allowed (that is the point), and unions do not disable enums.
Tags: D4 | S6 | Advanced | nullable-union-limit-interaction

### Q30. To represent "field genuinely not present in the source" under strict outputs while staying within constraints, which design is cleanest?

A) Mark every field `required` and instruct the model to guess
B) Use a nullable type (or a sentinel `enum` value like `"NOT_FOUND"`) so the model has a typed honest "absent" path, keeping `required` only for fields that truly must always exist
C) Omit the field from the schema and parse it from prose
D) Use a free-text field and normalize later
Answer: B
Explanation: A typed absent path — nullable or a sentinel enum — lets the model report "not present" instead of fabricating, while you reserve `required` for genuinely-mandatory fields. Forcing a guess causes hallucination; prose parsing and free-text reintroduce fragility and drift.
Tags: D4 | S6 | Intermediate | sentinel-vs-required-absent

### Q31. A 5-level deeply nested strict schema for a contract keeps hitting grammar-size/complexity limits and extracts unreliably. What is the recommended restructuring?

A) Raise the retry cap and keep the deep object
B) Flatten the structure and/or split genuinely separate concerns (e.g. header vs line items) into separate strict tools or passes, reducing compiled-grammar size and error rate
C) Convert to prose output and regex it later
D) Remove `additionalProperties: false` to shrink the grammar
Answer: B
Explanation: Deep nesting raises error rates and compiled-grammar size; flattening and splitting separate concerns into separate tools/passes is the documented move (and respects strict complexity limits). A higher cap masks the problem, prose reintroduces parsing, and removing `additionalProperties: false` weakens the contract rather than helping reliability.
Tags: D4 | S6 | Advanced | flatten-split-grammar-size

## Section H — Mixed Advanced Integration

### Q32. Which end-to-end S6 design is correct on the *current* API, accounting for what strict mode does and does not guarantee?

A) Prefill `{`, parse the prose, catch exceptions by returning `{}`
B) Use JSON outputs via `output_config` (or a `strict: true` tool) for guaranteed shape; validate completeness (truncation), ranges/patterns (post-generation), and business semantics; feed specific diagnostics back on a bounded corrective retry; escalate exhausted items to a dead-letter queue; upsert on a stable key; and evaluate per slice
C) Maximize the retry cap and trust whatever decodes
D) Raise `effort` to maximum on every request and skip validation
Answer: B
Explanation: The current-API production pattern uses strict outputs for shape, then explicitly validates the things strict mode does NOT cover (completeness, ranges/patterns, semantics), with specific corrective retries, bounded caps, dead-letter escalation, idempotent upserts, and per-slice eval. Option A chains retired/anti-pattern moves (last-turn prefill is unsupported on current models; silent suppression), and C/D skip the validation that strict mode does not replace.
Tags: D4 | S6 | Advanced | current-api-end-to-end

### Q33. A team migrating from an older model removes their prefill-based JSON forcing and adds `output_config` JSON outputs, but keeps `try: parse() except: return {}` around the call "just in case." What is the remaining problem?

A) Nothing; the empty-object fallback is now safe because strict mode guarantees shape
B) The `except: return {}` is still silent suppression (anti-pattern #7): truncation, transport errors, or post-validation failures can still raise, and swallowing them as an empty success hides real failures — surface, retry, or escalate instead
C) `output_config` cannot coexist with try/except
D) The fallback should return `null` instead of `{}` to be correct
Answer: B
Explanation: Strict shape does not make every failure mode disappear (truncation, network errors, semantic/range validation), so swallowing exceptions into `{}` remains silent suppression that fakes success. The fix is to surface/retry/escalate; whether the placeholder is `{}` or `null` is irrelevant — the suppression itself is the bug.
Tags: D4 | S6 | Advanced | suppression-survives-strict

### Q34. For a high-volume extraction service on the current API, which placement maximizes prompt-cache reuse while keeping per-request flexibility?

A) Put the persona, schema, and stable examples in each user turn so they can vary
B) Put persona, standing constraints, schema, and stable few-shot examples in the system prompt (stable, cacheable across requests) and put only the per-request document/task in the user turn
C) Put everything in a prefilled assistant turn
D) Put the schema in the system prompt but the examples in a prefill
Answer: B
Explanation: Stable system-prompt content (persona, constraints, schema, fixed examples) is the cacheable prefix; per-request data goes in the user turn. This split sharpens accuracy and maximizes cache reuse. Prefill is not a viable home on current models (last-turn prefill unsupported), and varying stable content per user turn defeats caching.
Tags: D4/D5 | S6 | Intermediate | system-cache-placement-current

### Q35. Anthropic recommends roughly how many few-shot examples for best results, and what does the doc additionally suggest you can do with them?

A) Exactly one; never more
B) Around 3–5 diverse examples; you can also ask Claude to evaluate your examples for relevance and diversity, or to generate additional ones from your initial set
C) 50+ to approximate fine-tuning
D) Zero, because examples bias the model
Answer: B
Explanation: The documented guidance is 3–5 examples for best results, and notably you can have Claude critique your examples for relevance/diversity or generate more from a seed set. One example underspecifies; dozens are unnecessary; examples are a primary steering tool, not a bias hazard.
Tags: D4 | S6 | Foundational | fewshot-3to5-and-selfcritique

### Q36. On a current adaptive-thinking model, a developer adds extended thinking with a large `budget_tokens` to "be safe" on every request in a high-volume extraction — what is wrong with this?

A) Correct; more thinking budget is always safer
B) Wasteful and outdated: `budget_tokens` extended thinking is deprecated, thinking adds latency/cost and should be reserved for genuinely hard items, and current models use adaptive thinking driven by `effort` + complexity (often answering simple items directly)
C) It will return invalid JSON
D) It disables the schema
Answer: B
Explanation: Blanket high-budget thinking on every simple request is exactly what adaptive thinking is designed to avoid — it adds latency/cost for no gain, and `budget_tokens` is deprecated in favor of `effort`-driven adaptive thinking that responds directly to easy queries. It does not invalidate JSON or disable the schema, but it is the wrong default.
Tags: D4 | S6 | Advanced | blanket-thinking-antipattern

### Q37. A reviewer claims "strict structured outputs and forced tool_use give the same guarantee." On the current API, what is the precise distinction worth knowing?

A) They are identical in every respect
B) `strict: true` on a tool grammar-constrains the tool's *inputs* by construction; a plain forced `tool_choice` without `strict` strongly steers but does not grammar-guarantee conformance — and `output_config` JSON outputs constrain the *text* response. Match the mechanism to whether you are constraining a tool input or the text reply
C) Forced tool_use guarantees semantic correctness; strict outputs do not
D) Strict outputs only work in batch mode
Answer: B
Explanation: Forcing a tool (`tool_choice`) makes the model call it, but conformance is only *grammar-guaranteed* when `strict: true` is set; `output_config` JSON outputs handle the text-response surface. None of these guarantee *semantics*. The distinction (which surface, and strict vs merely forced) is the exam-relevant nuance.
Tags: D4 | S6 | Advanced | strict-tool-vs-forced-vs-json-outputs

### Q38. An architect needs range/pattern guarantees that the strict decoder cannot provide, and absolute certainty no out-of-range value reaches the database. What layered design satisfies this?

A) Rely solely on strict mode and skip validation
B) Use strict outputs for shape, keep the range/pattern rules in the schema/descriptions, and add a post-generation validation gate that rejects out-of-range values into the corrective-retry loop before any database write
C) Move the ranges into the system prompt and trust the model
D) Widen the schema to accept any number and clamp later
Answer: B
Explanation: Because ranges/patterns are not decode-time guaranteed, certainty comes from a validation gate after generation that feeds violations back as specific corrective retries before committing. Strict-only skips the needed check, trusting the prompt is not a guarantee, and clamping silently alters data (hiding the failure) rather than surfacing it.
Tags: D4 | S6 | Advanced | layered-range-guarantee
