# Domain 4 — Prompt Engineering & Structured Output: Practice Questions

These scenario-based practice questions target **Domain 4 (Prompt Engineering & Structured Output, 20%)** of the Claude Certified Architect – Foundations (CCA-F) exam. They cover clear-and-direct prompting, multishot/few-shot, chain-of-thought, XML tags, prefill, prompt chaining, long-context ordering, hallucination reduction, the three structured-output approaches (`tool_use` JSON schema vs strict structured outputs vs prefill), the validation-and-retry loop, and schema design. Most items are anchored to **S6 Structured Data Extraction**, with several to **S2 Code Generation with Claude Code** and **S5 Claude Code for CI/CD**.

This is original study material grounded in publicly documented Anthropic behavior (prompt engineering, tool use, and structured outputs at https://platform.claude.com/docs and https://docs.anthropic.com/en/docs/build-with-claude). It is NOT a dump of live or proctored exam questions. Where an exact API value is uncertain, the behavior is described conceptually rather than invented.

Each question is tagged with its difficulty and anchoring scenario on a `Tags:` line under the explanation.

## Section 1 — Clear and Direct Prompting

### Q1. A Claude Code prompt for S2 generates a date-formatting helper that is correct but uses an inconsistent casing convention and omits the timezone edge case; the team keeps adding "VERY IMPORTANT: be careful!!!" to the prompt with no improvement. What is the best fix?
A) Add more emphatic, all-caps emphasis and exclamation marks to stress the requirements
B) Add explicit, specific instructions: the exact format, the timezone edge-case rule, and a worked example of the desired output
C) Switch the request to a higher temperature so the model explores more variations
D) Tell the model "do not make mistakes" at the end of the prompt
Answer: B
Explanation: Vague prompts cause "wrong but confident" output; the documented fix is specificity and structure (explicit format, edge-case rules, examples), not louder pleading. Option A is the classic emphatic-prose anti-pattern that does not add information. Higher temperature increases variance, the opposite of what consistency needs.
Tags: D4 | S2 | Foundational | clear-and-direct

### Q2. Why does Anthropic recommend treating Claude like "a brilliant new hire with no shared context" when writing instructions?
A) Because Claude cannot follow more than one instruction at a time
B) Because spelling out the goal, audience, format, edge-case handling, and what NOT to do removes the ambiguity that produces confidently wrong answers
C) Because new hires require lower temperature settings
D) Because it forces you to use chain-of-thought on every request
Answer: B
Explanation: The mental model exists to push you toward explicit context: goal, audience, format, edge cases, and exclusions. Ambiguity is the leading cause of plausible-but-wrong output. The other options misstate the rationale or invent unrelated mechanics.
Tags: D4 | S2 | Foundational | clear-and-direct

### Q3. A prompt asks Claude to "summarize the report" but reviewers complain summaries vary wildly in length, tone, and which sections they cover. Which change most directly fixes this?
A) State the exact length, the target audience, the required sections to cover, and the tone in the instructions
B) Increase the model size and hope longer reasoning stabilizes the format
C) Add the word "consistently" to the instruction
D) Run the same prompt five times and pick the best output manually
Answer: A
Explanation: Output variance from an under-specified task is solved by specifying the success criteria (length, audience, sections, tone). A bigger model does not fix an ambiguous spec. Adding "consistently" is emphatic prose with no concrete constraint, and manual cherry-picking is not a production fix.
Tags: D4 | S2 | Foundational | clear-and-direct

### Q4. When giving Claude a multi-step procedure, which formatting choice most improves reliable step-by-step execution?
A) A single dense paragraph describing all steps at once
B) Numbered sequential steps with explicit success criteria the model can self-check against
C) Only the final desired outcome, letting the model infer the steps
D) Emphatic adjectives on each requirement
Answer: B
Explanation: Anthropic recommends numbered, sequential steps and explicit success criteria so the model can verify its own work as it proceeds. A dense paragraph and "infer the steps yourself" both reintroduce ambiguity, and emphasis adds no structure.
Tags: D4 | S2 | Foundational | clear-and-direct

## Section 2 — Multishot / Few-Shot Examples

### Q5. In S6, an extraction pipeline returns the status field as "Open", "OPEN", "in progress", and "pending-open" across documents. The content is right but the formatting is inconsistent. What is the single most effective fix?
A) Add a longer English paragraph describing the exact format you want
B) Constrain the field to an `enum` of allowed values (and ideally show one example each)
C) Normalize the strings with regex in application code after extraction
D) Raise the temperature so the model considers more options
Answer: B
Explanation: A closed categorical field with formatting drift is solved by an `enum` that forces the model to pick from a fixed vocabulary, not by post-hoc string cleanup or a longer description. Regex normalization treats the symptom downstream instead of fixing the contract; higher temperature worsens drift.
Tags: D4 | S6 | Intermediate | enum-vs-normalization

### Q6. How many diverse examples does Anthropic generally recommend as the sweet spot for locking in output format and edge-case handling with few-shot prompting?
A) Exactly one perfect example
B) Around 3–5 diverse, relevant examples that cover tricky cases, not just the happy path
C) At least 50 examples to approximate fine-tuning
D) Zero — examples bias the model
Answer: B
Explanation: Anthropic documents roughly 3–5 diverse examples as the reliable range for pinning format and edge-case handling. One example underspecifies the pattern; dozens are unnecessary and costly. Examples are the single most reliable way to teach the shape of an answer, not a source of harmful bias.
Tags: D4 | S6 | Foundational | multishot

### Q7. When supplying few-shot examples, why wrap each example in tags such as `<example>...</example>`?
A) Tags are required syntax or the API rejects the request
B) So the model can clearly distinguish demonstration examples from the live input it must actually process
C) Tags automatically lower the token cost of examples
D) Tags force the model into JSON mode
Answer: B
Explanation: Wrapping examples in tags helps the model separate demonstrations from the real input, reducing the chance it treats an example as the task. Tags are a convention Claude is tuned to respect, not a hard API requirement, and they do not change token cost or trigger a JSON mode.
Tags: D4 | S6 | Intermediate | multishot-xml

### Q8. A few-shot extraction prompt shows only clean, complete invoices as examples, and the pipeline fails on invoices with missing totals or multiple line-item tables. What is the best correction?
A) Remove the examples entirely so the model generalizes freely
B) Add examples that demonstrate the tricky cases — missing fields, ambiguity, multiple matches — with the correct handling for each
C) Add a single emphatic sentence: "Handle ALL edge cases correctly"
D) Increase the iteration cap on the retry loop
Answer: B
Explanation: Few-shot examples teach edge-case handling only if they include the edge cases; happy-path-only examples leave the tricky slices unspecified. Removing examples removes the format anchor entirely, emphatic prose adds nothing, and a higher retry cap does not teach the model how to handle the case.
Tags: D4 | S6 | Intermediate | multishot-edge-cases

### Q9. Few-shot examples are the highest-leverage fix for which class of failure?
A) The model reasons incorrectly through a multi-hop calculation
B) The model produces the right content but in an inconsistent format (casing, date format, null handling, ordering)
C) The model hits the token length limit mid-output
D) The model lacks permission to call a tool
Answer: B
Explanation: Examples primarily fix *format* inconsistency by demonstrating the exact target shape and style. A reasoning error is better addressed with chain-of-thought or extended thinking; truncation and tool permissions are unrelated mechanics. This format-vs-reasoning distinction is a recurring exam trap.
Tags: D4 | S6 | Intermediate | multishot-vs-reasoning

## Section 3 — Chain-of-Thought (CoT)

### Q10. In S2, Claude rushes to a wrong answer on a tricky algorithmic transformation that requires several reasoning steps. Which technique most directly improves accuracy here?
A) Add three more happy-path few-shot examples of the final output
B) Let the model reason step by step before answering, structuring the thinking ("First… Second…")
C) Lower the max tokens so it commits faster
D) Add "be accurate" to the system prompt
Answer: B
Explanation: Chain-of-thought is the documented lever for multi-step reasoning where the model would otherwise rush to a wrong answer. More happy-path examples fix format, not reasoning. Reducing max tokens cuts off the reasoning room, and emphatic prose adds nothing.
Tags: D4 | S2 | Intermediate | chain-of-thought

### Q11. When you need clean parseable output but also want the model to reason first, what is the recommended way to handle the reasoning text?
A) Put the reasoning in the same JSON field as the final answer
B) Isolate the reasoning in a separate `<thinking>` block (or use extended thinking) so it can be stripped, keeping the structured answer clean
C) Ask the model to skip reasoning entirely to keep output clean
D) Discard the structured output and parse the reasoning prose with regex
Answer: B
Explanation: Keep scratch reasoning in a dedicated `<thinking>` block (or the extended-thinking channel) so the final JSON stays clean and parseable. Mixing reasoning into the answer field breaks parsing — a documented trap. Skipping reasoning sacrifices the accuracy you wanted; regex-parsing prose is fragile.
Tags: D4 | S2 | Intermediate | cot-thinking-separation

### Q12. For which task is chain-of-thought generally NOT worth the added latency and tokens?
A) A complex multi-constraint classification
B) A trivial single-field lookup from a short document
C) A multi-hop arithmetic calculation
D) Resolving conflicting clauses in a contract
Answer: B
Explanation: CoT trades latency and tokens for accuracy and should be reserved for genuinely hard reasoning; a trivial lookup does not need it. The other options are exactly the multi-step reasoning cases where CoT (or extended thinking) pays off.
Tags: D4 | S6 | Foundational | cot-when-not-to-use

### Q13. What is the core trade-off of using chain-of-thought prompting?
A) It guarantees schema-valid JSON in exchange for higher cost
B) It improves accuracy on multi-step problems at the cost of more latency and tokens
C) It eliminates the need for a validation-and-retry loop
D) It removes the need for examples
Answer: B
Explanation: CoT spends more tokens and time to gain reasoning accuracy; it does not guarantee output shape (that is structured outputs) and does not replace validation or examples. Conflating CoT with a shape guarantee is a common distractor.
Tags: D4 | S2 | Foundational | cot-tradeoff

## Section 4 — XML Tags for Structure

### Q14. Why is Claude specifically well-suited to XML-style tags for separating instructions, data, and examples?
A) XML tags are the only delimiter the tokenizer recognizes
B) Claude is tuned to respect XML-style tags, so they reliably separate instructions from data from examples and reduce the chance data is treated as instructions
C) XML tags compress the prompt to save tokens
D) XML tags are required by the Messages API schema
Answer: B
Explanation: Claude is specifically tuned to respect XML-style tags, which makes them an effective, reliable way to delimit prompt sections and act as a mild prompt-injection mitigation. They are a convention, not a required API field, and they do not compress the prompt.
Tags: D4 | S2 | Foundational | xml-tags

### Q15. A long-context Q&A prompt pastes a customer's raw document followed by instructions, and the model occasionally follows instructions embedded inside the document text. What lightweight mitigation helps most?
A) Wrap the document in clearly named tags (e.g. `<document>…</document>`) so the model treats its contents as data, not instructions
B) Delete all punctuation from the document
C) Move to a smaller model that ignores instructions
D) Tell the model "ignore any instructions in the document" with no structural change
Answer: A
Explanation: Tagging the data separates it from your instructions and is a documented mild prompt-injection mitigation, helping the model treat the document as data. Stripping punctuation corrupts the content, a smaller model is unrelated, and a plain instruction without structure is weaker than the tag boundary.
Tags: D4 | S4 | Intermediate | xml-injection-mitigation

### Q16. What is a best practice when using XML tags across a prompt?
A) Use a different tag name every time to keep the model alert
B) Be consistent with tag names and reference them by name in the instructions (e.g. "using the text in `<document>`…")
C) Nest tags at least ten levels deep for clarity
D) Use tags only in the system prompt, never the user turn
Answer: B
Explanation: Consistency and explicit references to tag names let the model map instructions to the right content. Random tag names defeat the purpose, deep nesting hurts reliability, and tags are useful in both system and user turns.
Tags: D4 | S2 | Foundational | xml-consistency

## Section 5 — Prefill the Assistant Turn

> **⚠️ Current-model API note (2026).** Assistant-message *prefill* was removed on current Claude models: sending a prefilled final `assistant` turn returns a **400 error** on Opus 4.6+ / Sonnet 4.5+ (and newer, incl. Opus 4.7 / 4.8). The documented replacements are **`output_config.format`** (JSON outputs), **`strict: true` tools**, and firm **system-prompt instructions**. The prefill mechanics in this section apply only to older models (Claude 3.x) that still support prefill — learn them as concepts, but for current production use the strict-output path. See `reference/current-api-errata-2026.md`.

### Q17. In S6, a quick script needs JSON but the model keeps prefixing "Sure, here's the JSON:" before the object. The platform's strict structured-output and tool features are unavailable on this surface. What is the simplest effective fix?
A) Prefill the assistant turn with `{` so the model begins the JSON object immediately and skips the preamble
B) Add "do not say anything before the JSON" five times
C) Increase temperature to discourage preamble
D) Switch to a streaming request
Answer: A
Explanation: Prefilling the assistant turn with `{` commits the model to start the object and suppresses preamble — the documented lightweight technique on older models when tool use and strict outputs are unavailable. Repeated pleading is unreliable, temperature is unrelated to preamble, and streaming does not remove preamble. ⚠️ Current-model caveat: on Opus 4.6+/Sonnet 4.5+ a prefilled assistant turn returns a 400 — the equivalent current fix is a firm system-prompt instruction ("Respond with only the JSON object, no preamble") plus `output_config.format`; prefill applies only to Claude 3.x.
Tags: D4 | S6 | Intermediate | prefill

### Q18. A developer prefills the assistant turn with `{`, parses the model's continuation, and gets invalid JSON because they forgot something. What was the likely cause?
A) Prefill is not allowed for JSON
B) The model continues *after* the prefilled `{`, so the opening brace must be reattached when reconstructing the full object
C) The model always omits the closing brace when prefilled
D) Prefill forces XML, not JSON
Answer: B
Explanation: When you prefill `{`, the model's output continues after it, so you must reattach the brace to reconstruct a complete object. Prefill is allowed for JSON, does not force XML, and does not categorically drop closing braces; pairing it with a stop sequence further bounds the output.
Tags: D4 | S6 | Intermediate | prefill-reattach-brace

### Q19. Which statement about prefill is correct for the exam?
A) Prefill is the strongest structured-output mechanism and guarantees schema-valid JSON
B) Prefill is a formatting lever that shapes the start of the response but provides no schema-conformance guarantee, so output still needs validation
C) Prefill content may end with trailing whitespace to separate it from the model's output
D) Prefill replaces the need for a tool schema in all cases
Answer: B
Explanation: Prefill is a formatting convenience, not a conformance guarantee — the model can still emit malformed or extra-field JSON, so you validate. Prefill content is not allowed to end with trailing whitespace, and it does not replace tool schemas or strict structured outputs as the stronger contracts. ⚠️ Note: prefill is also unsupported on current models (400 on Opus 4.6+/Sonnet 4.5+) — prefer `output_config.format` / `strict: true` tools today.
Tags: D4 | S6 | Intermediate | prefill-not-a-guarantee

### Q20. To make a prefilled-JSON response halt cleanly with no trailing commentary, what should you pair the prefill with?
A) A higher temperature
B) A stop sequence that halts generation at the end of the object
C) A second prefill block
D) Extended thinking
Answer: B
Explanation: Pairing prefill with a stop sequence bounds generation so it stops at the close of the object, avoiding trailing prose. Temperature does not bound output, a second prefill is not a mechanism, and extended thinking adds reasoning, not a clean stop.
Tags: D4 | S6 | Intermediate | prefill-stop-sequence

## Section 6 — Prompt Chaining

### Q21. Why does Anthropic recommend decomposing a complex job into a chain of focused prompts rather than one mega-prompt?
A) Chaining always uses fewer total tokens
B) Each prompt has a single responsibility, which improves accuracy and debuggability and enables a fresh-context review pass
C) Chaining removes the need to validate any output
D) The Messages API forbids long single prompts
Answer: B
Explanation: Prompt chaining isolates responsibilities, which improves accuracy and makes failures easier to localize, and it enables a clean review/critique pass on fresh context. It does not necessarily reduce total tokens, does not remove validation, and there is no API ban on long prompts.
Tags: D4 | S5 | Intermediate | prompt-chaining

### Q22. Prompt chaining is the prompt-level analogue of which higher-level pattern?
A) Increasing the iteration cap on an agent loop
B) Multi-agent decomposition with a fresh-context review pass (as in D1/S5)
C) Sentiment-based escalation
D) Self-reported confidence scoring
Answer: B
Explanation: Decomposing into focused prompts mirrors multi-agent decomposition, including the fresh-context critique that avoids same-session review bias. The other options are unrelated mechanics or listed anti-patterns (confidence-driven escalation, sentiment escalation).
Tags: D4 | S5 | Intermediate | chaining-vs-multiagent

### Q23. In S6, an extract-then-critique pipeline runs the critique step inside the same conversation that produced the extraction, and the critique rarely catches the extractor's own mistakes. What is the documented reason and fix?
A) The reason is temperature; lower it on the critique step
B) The critique inherits the author's reasoning bias from the shared context; run the review in a fresh context (a separate prompt/agent)
C) The reason is token limits; shorten the document
D) The critique needs a self-reported confidence score to decide what to recheck
Answer: B
Explanation: Same-session self-review inherits the author's reasoning bias (anti-pattern #9); a fresh-context review pass is the fix and is exactly what prompt chaining enables. Temperature and token limits are not the cause, and self-reported confidence is a separate anti-pattern, not a fix.
Tags: D4 | S6 | Advanced | fresh-context-review

## Section 7 — Long-Context Ordering

### Q24. In S4, a prompt pastes a 50-page contract and places the question at the very top, before the document. Accuracy on long-context questions is poor. What reordering does Anthropic recommend?
A) Keep the question first; long inputs should always lead with the ask
B) Put the large document near the top and the query/instructions at the bottom
C) Split the document into 50 separate requests
D) Move both the document and the question into the system prompt
Answer: B
Explanation: For long inputs (roughly 20K+ tokens), Anthropic documents placing the long-form content near the top and the query at the bottom, which can meaningfully improve quality. Splitting into 50 requests loses cross-document context, and stuffing everything into the system prompt does not address the ordering issue.
Tags: D4 | S4 | Intermediate | long-context-ordering

### Q25. Beyond ordering, which two techniques improve grounded Q&A quality over long documents?
A) Wrap each document in tags with metadata (source, index), and ask the model to quote relevant passages first, then answer
B) Raise the temperature and remove the schema
C) Forbid the model from saying "I don't know"
D) Repeat the question at the start, middle, and end
Answer: A
Explanation: Per-document tags with metadata plus quote-first grounding are the documented long-context grounding aids. Higher temperature and no schema reduce reliability, forbidding "I don't know" forces fabrication, and repeating the question does not improve grounding.
Tags: D4 | S4 | Intermediate | long-context-grounding

### Q26. Quote-first grounding (ask the model to extract relevant passages before answering) primarily helps by what mechanism?
A) It guarantees the JSON schema validates
B) It anchors the answer to evidence in the source, reducing ungrounded fabrication
C) It reduces token usage
D) It removes the need for citations entirely
Answer: B
Explanation: Quoting the relevant source passages first grounds the conclusion in evidence, reducing hallucination on long-context tasks. It does not guarantee schema shape, generally increases tokens slightly, and is itself a form of citation rather than a replacement for one.
Tags: D4 | S4 | Intermediate | quote-first-grounding

## Section 8 — Reducing Hallucinations

### Q27. In S6, a schema marks every field `required` and the prompt demands a value for each, even when data is genuinely absent from the document. What failure does this design cause?
A) It causes truncated output
B) It forces the model to fabricate values for absent fields to satisfy the required constraint
C) It guarantees schema-valid but slower output
D) It triggers a tool-permission error
Answer: B
Explanation: Requiring a value with no null/"not found" path forces fabrication — a documented hallucination cause. The fix is a typed "absent" path (nullable field or a sentinel enum value). It is not about truncation, latency, or tool permissions.
Tags: D4 | S6 | Intermediate | hallucination-null-path

### Q28. Which combination best reduces hallucination in a grounded factual extraction task?
A) Give an explicit "I don't know"/`null` out, require quotes from the source, lower temperature, and constrain output with a schema
B) Raise temperature, forbid "I don't know", and ask for confidence scores
C) Add emphatic instructions not to hallucinate
D) Increase the model size only
Answer: A
Explanation: The documented hallucination-reduction stack is: permit "I don't know"/`null`, require source citations, lower temperature for factual work, and constrain output with a schema. Forbidding "I don't know" and demanding confidence scores both push toward fabrication (and self-reported confidence is an anti-pattern). Emphasis and a bigger model alone are not the documented fixes.
Tags: D4 | S6 | Intermediate | hallucination-stack

### Q29. For genuinely-absent optional data, what is the recommended schema treatment?
A) Make the field `required` and instruct the model to guess the most likely value
B) Make the field non-required or include an explicit nullable type / sentinel so the model can honestly report "absent"
C) Omit the field from the schema and parse it from prose
D) Use a free-text field and normalize later
Answer: B
Explanation: A typed "unknown/not present" path (non-required, nullable, or a sentinel) lets the model say "absent" instead of fabricating. Forcing a guess causes hallucination, omitting the field reintroduces prose parsing, and free-text reintroduces formatting drift.
Tags: D4 | S6 | Foundational | schema-null-sentinel

## Section 9 — Structured Output: Choosing the Approach

### Q30. In S6, an extraction service string-parses Claude's prose reply with regex and breaks whenever the model rephrases its answer. What is the canonical fix?
A) Add more few-shot examples of the prose format
B) Define a tool whose `input_schema` is the desired shape and force it via `tool_choice`, reading structure from `tool_use.input`
C) Add stronger instructions telling the model never to rephrase
D) Lower the temperature so phrasing is stable
Answer: B
Explanation: Using a tool's `input_schema` as a typed output contract (forced via `tool_choice`) removes prose-parsing fragility — the canonical S6 fix. More examples or stricter phrasing instructions still leave you parsing prose, and lower temperature does not make regex-on-prose robust.
Tags: D4 | S6 | Intermediate | tool-use-schema

### Q31. When you set `tool_choice` to force a single extraction tool such as `record_extraction`, what behavior are you guaranteeing?
A) The tool will be executed by Anthropic's servers automatically
B) Claude must emit a `tool_use` block for that tool rather than replying in prose, returning input conforming to the schema's shape
C) The returned values are guaranteed semantically correct
D) The model will refuse if the data is ambiguous
Answer: B
Explanation: Forcing the specific tool makes Claude respond with a `tool_use` call for it instead of prose; you read `tool_use.input`. The tool is a typed envelope you do not have to execute, the platform does not run it for you, and forced tool use guarantees shape, not semantic correctness.
Tags: D4 | S6 | Intermediate | tool-choice-forced

### Q32. A CI gate in S5 needs downstream code to consume Claude's output with zero parsing tolerance and a hard guarantee the JSON validates against the schema every time. Which approach is best where the surface supports it?
A) Prefill `{` and hope the model stays in shape
B) Strict structured outputs / an output schema that constrains decoding to guarantee schema-conformant JSON, with a tool schema as the fallback
C) Parse the prose reply and add a regex cleanup pass
D) Add more few-shot examples until it stops failing
Answer: B
Explanation: Strict structured outputs constrain generation so the result is guaranteed to match the schema — the strongest contract, ideal for a zero-tolerance CI gate, with tool schemas as fallback where unsupported. Prefill gives no guarantee, prose parsing is the failure mode, and examples reduce drift but never guarantee shape.
Tags: D4 | S5 | Advanced | strict-structured-outputs

### Q33. Which ordering ranks the three structured-output approaches from strongest to weakest guarantee of correct JSON shape?
A) Prefill `{` > tool_use schema > strict structured outputs
B) Strict structured outputs > tool_use with `input_schema` + forced `tool_choice` > prefill `{` + stop sequence
C) Tool_use schema > strict structured outputs > prefill
D) They are all equally strong; choose by preference
Answer: B
Explanation: The reliability ladder runs strict structured outputs (guaranteed by constrained decoding) > tool-use schema (strongly encouraged, broadly supported) > prefill + stop sequence (convenience, no guarantee). The other orderings invert the ladder or deny that strength differs.
Tags: D4 | S6 | Intermediate | structured-output-ladder

### Q34. What is the key difference between a `tool_use` `input_schema` and strict structured outputs?
A) There is none; both guarantee schema conformance identically
B) Strict structured outputs enforce conformance during decoding (guaranteed shape), whereas a tool schema strongly encourages but does not strictly guarantee conformance on every surface
C) Tool schemas guarantee semantic correctness, strict outputs do not
D) Strict outputs only work for XML, tool schemas only for JSON
Answer: B
Explanation: Strict structured outputs constrain the decoder so output is valid by construction; a tool `input_schema` shapes and strongly steers output but is not the same hard guarantee everywhere. Neither guarantees semantics, and the XML/JSON claim is fabricated.
Tags: D4 | S6 | Advanced | tooluse-vs-strict

### Q35. Which statement about availability of strict structured outputs is correct for the exam?
A) Strict structured outputs are available on every model and SDK surface
B) Strict structured outputs are a specific capability; confirm your model and SDK surface support it before designing around it, and fall back to a tool schema if not
C) Strict structured outputs are deprecated in favor of prefill
D) Strict structured outputs only matter for streaming
Answer: B
Explanation: Strict structured outputs are a specific feature you must confirm is supported on your target model/SDK; the documented fallback is a tool `input_schema`. They are not universal, not deprecated, and not streaming-specific.
Tags: D4 | S5 | Intermediate | strict-availability

### Q36. Even with strict structured outputs guaranteeing the JSON shape, why must you still validate?
A) Because strict mode often produces invalid JSON
B) Because structural validity is not semantic correctness — an extracted date can be schema-valid but the wrong date, and an enum choice can be the wrong category
C) Because validation reduces token cost
D) Because strict mode disables the schema after the first request
Answer: B
Explanation: Strict outputs guarantee shape, not meaning; you still validate value ranges, cross-field rules, and business correctness. Strict mode does not produce invalid JSON, does not cut token cost, and does not disable the schema.
Tags: D4 | S6 | Advanced | shape-vs-semantics

### Q37. For simple, low-stakes formatting where neither tool use nor strict outputs are needed, which approach is appropriate — always paired with what?
A) Prefill `{` + stop sequence, always paired with a parse-and-validate step
B) Prefill `{` alone, with no validation needed because it is low-stakes
C) Strict structured outputs, paired with sentiment analysis
D) A larger model, paired with higher temperature
Answer: A
Explanation: Prefill `{` plus a stop sequence is the lightweight convenience approach, but because it gives no conformance guarantee it must still be paired with parse-and-validate. "No validation needed" is the trap; strict outputs and bigger models are overkill or unrelated.
Tags: D4 | S6 | Foundational | prefill-plus-validate

## Section 10 — The Validation-and-Retry Loop

### Q38. In S6, extraction "mostly works but occasionally returns broken JSON." Which architecture is correct?
A) Increase temperature so the model varies its output
B) Validate the output, feed the specific validation error back to the model for a corrected retry, bound retries with a small cap, then escalate to human review on exhaustion
C) Catch the parse exception and return an empty object so the pipeline keeps running
D) Filter out the rows that fail to parse and report success
Answer: B
Explanation: The documented reliable pattern is validate → feed the specific error back → corrected retry → bounded cap → escalate. Returning `{}` on exception is silent suppression (anti-pattern #7), filtering bad rows hides the failure, and higher temperature increases breakage.
Tags: D4 | S6 | Intermediate | validation-retry-loop

### Q39. What makes a corrective retry message effective?
A) A generic "that was wrong, try again"
B) A specific, machine-derived error naming the exact failing field, the rule that broke, and the offending value (e.g. "field `total` must be a number, got string 'N/A'")
C) A request to lower the temperature
D) Re-sending the identical original prompt with no new information
Answer: B
Explanation: A targeted error message lets the model self-correct precisely in one pass; the corrective signal must be specific and machine-derived. A generic "try again" wastes a turn, temperature is irrelevant to the contract, and re-sending the same prompt adds no information.
Tags: D4 | S6 | Intermediate | corrective-retry-specific

### Q40. A pipeline does `try: parse(...) except: return {}`. Which anti-pattern is this, and why is it wrong?
A) It is fine because empty results are valid
B) It is silently suppressing the error (anti-pattern #7) — returning an empty result as if successful hides data-quality failures from downstream systems and humans
C) It is the iteration-cap anti-pattern
D) It is sentiment-based escalation
Answer: B
Explanation: Returning `{}` on a parse failure is silent suppression: it masks the failure as a success, hiding it from the system and any human in the loop. The correct behavior is to surface the error, retry with the specific diagnostic, and fail loudly. The other anti-patterns named do not match this pattern.
Tags: D4 | S6 | Intermediate | silent-suppression

### Q41. Why is feeding back a generic "validation failed" message instead of the concrete diagnostic an anti-pattern?
A) It uses too many tokens
B) Generic error messages hide the diagnostic context (which field, which rule, the offending value) the model needs to self-correct (anti-pattern #6)
C) It violates the schema
D) It forces extended thinking
Answer: B
Explanation: A generic error starves the model of the context required to fix the specific problem (anti-pattern #6); the right move is to echo the exact failing field, rule, and value. It is not about token count, schema validity, or extended thinking.
Tags: D4 | S6 | Intermediate | generic-error-antipattern

### Q42. On retry-cap exhaustion in an extraction loop, what is the correct terminal behavior?
A) Silently accept whatever the last attempt produced
B) Surface a real error with diagnostic context and route to human review or a dead-letter queue — never a fake success
C) Return an empty result and mark the job successful
D) Loop forever until it eventually parses
Answer: B
Explanation: The cap is a safety backstop; on exhaustion you must fail loudly with diagnostics and escalate (human review or dead-letter), never fake success. Silent acceptance and empty-as-success are suppression anti-patterns, and an unbounded loop defeats the cap's purpose.
Tags: D4 | S6 | Intermediate | terminal-failure-escalation

### Q43. Which of the following is the WEAKEST output contract and should never be the only safeguard in a production extraction pipeline?
A) Strict structured outputs with post-validation
B) A polite instruction asking for JSON, with no prefill, no stop sequence, and no schema, and no validation
C) A tool `input_schema` with forced `tool_choice` plus validation
D) Prefill `{` + stop sequence + schema-in-prompt + validation
Answer: B
Explanation: A bare "please return JSON" with no prefill, stop sequence, schema, or validation is the weakest contract and breaks under model or prompt drift. The other three are progressively stronger and all include validation, which is the production requirement.
Tags: D4 | S6 | Foundational | weakest-contract

### Q44. The retry cap in the validation loop should be understood as what?
A) The primary control that decides when extraction is correct
B) A small bounded safety backstop, not the primary correctness mechanism — validation and the corrective retry are
C) A reason to skip validation
D) A confidence threshold
Answer: B
Explanation: The cap is a backstop (echoing anti-pattern #2: arbitrary caps must not be the primary control). Correctness comes from validation plus the specific corrective retry; on cap exhaustion you escalate. It does not replace validation or function as a confidence threshold.
Tags: D4 | S6 | Intermediate | retry-cap-backstop

## Section 11 — Schema Design for Extraction

### Q45. Which trio of schema features does the exam highlight as "hardening" an extraction schema?
A) `temperature`, `top_p`, `max_tokens`
B) `enum` (closed sets), `description` (inline instructions per field), and `required` (no quiet omissions)
C) `stream`, `stop_sequences`, `tool_choice`
D) `minimum`, `maximum`, `pattern` only
Answer: B
Explanation: `enum` + `description` + `required` is the documented hardening trio: closed vocabularies, per-field instructions, and mandatory fields. Sampling parameters are not schema features, and while `minimum`/`maximum`/`pattern` help where honored, they are not the named trio.
Tags: D4 | S6 | Foundational | schema-hardening-trio

### Q46. In a schema, what is the purpose of writing a `description` on each field?
A) It is decorative metadata the model ignores
B) Descriptions act as inline instructions the model reads (e.g. "ISO 8601 date; use the issue date, not the due date"), removing guesswork
C) Descriptions are required by JSON Schema or the request fails
D) Descriptions lower the temperature for that field
Answer: B
Explanation: Field descriptions are real prompt surface; the model reads them as inline extraction rules. They are not ignored, not strictly required by the API, and do not affect sampling parameters.
Tags: D4 | S6 | Foundational | schema-descriptions

### Q47. A status field returns "Open", "OPEN", "in progress", and "pending-open". The team plans to add a normalization function in application code. What does the exam consider the correct fix?
A) The normalization function — keep the schema as free text
B) Constrain the field to an `enum` of the allowed values so the model picks from a fixed vocabulary, making downstream switch logic total
C) Lower the temperature and keep free text
D) Add a confidence score to each status value
Answer: B
Explanation: The root-cause fix is an `enum` in the schema, not post-hoc string normalization in app code. Constraining the vocabulary eliminates the drift at the source and makes downstream code total. Free text plus normalization treats the symptom; confidence scores are an unrelated anti-pattern.
Tags: D4 | S6 | Intermediate | enum-root-cause

### Q48. Why does Anthropic recommend avoiding overly deep nesting in extraction schemas?
A) Deep nesting is rejected by the API
B) Deep, sprawling objects raise error rates and make partial-output recovery harder; flatter schemas extract more reliably and are easier to validate
C) Deep nesting always exceeds the token limit
D) Deep nesting disables enums
Answer: B
Explanation: Flatter schemas extract more reliably and recover better from partial output; deep nesting increases error rates. The API does not reject nesting outright, nesting does not categorically blow the token limit, and it does not disable enums.
Tags: D4 | S6 | Intermediate | schema-flat-vs-nested

### Q49. When a schema needs the narrowest possible type for a numeric quantity that cannot be fractional, which is preferred?
A) A free-text string field
B) `integer`, with documented format/units, plus `minimum`/`maximum` where the platform honors them
C) `number` with no constraints
D) `boolean`
Answer: B
Explanation: Use the narrowest type that fits (`integer` for whole quantities), document units, and add range constraints where honored. A string reintroduces parsing, unconstrained `number` is looser than needed, and `boolean` does not represent a quantity.
Tags: D4 | S6 | Intermediate | schema-narrow-types

### Q50. Two genuinely separate concerns (e.g., header fields and a long list of line items with complex rules) keep colliding in one giant schema, raising error rates. What is the recommended structural move?
A) Force everything into one deeply nested object and raise the retry cap
B) Split the genuinely separate concerns into separate tools/passes rather than one giant object
C) Drop the line-item rules to simplify
D) Switch to prose output and parse later
Answer: B
Explanation: When concerns are genuinely separate, splitting into separate tools or passes is more reliable than one sprawling object — the prompt-level analogue of decomposition. Forcing one giant object and bumping the cap masks the problem, dropping rules loses correctness, and prose output reintroduces parsing.
Tags: D4 | S6 | Advanced | schema-split-concerns

## Section 12 — Extended Thinking for Hard Reasoning

### Q51. In S6, an item requires resolving conflicting contract clauses with a multi-hop calculation, and accuracy is poor. Examples were already added and fixed only the format. What is the best next move?
A) Add more few-shot examples of the final output
B) Enable extended thinking to give the model a dedicated reasoning budget, then emit the final answer through a schema/tool
C) Raise the iteration cap on the retry loop
D) Increase temperature for more exploration
Answer: B
Explanation: Examples fix format; a hard reasoning step needs extended thinking (a dedicated reasoning budget), paired with a schema for the clean final answer. More examples will not fix the reasoning, a higher cap does not improve correctness, and higher temperature adds variance, not rigor.
Tags: D4 | S6 | Advanced | extended-thinking-reasoning

### Q52. Which statement about extended thinking is correct?
A) Extended thinking guarantees schema-valid JSON
B) Extended thinking is for reasoning depth, not formatting; pair it with a schema/tool for the final structured answer and keep reasoning out of the answer field
C) Extended thinking should be enabled on every request for safety
D) Extended thinking eliminates the need for validation
Answer: B
Explanation: Extended thinking adds reasoning depth via a token budget; it does not shape output, so you pair it with a schema and keep reasoning separate from the answer. It is not a shape guarantee, is reserved for hard slices (not every request, due to latency/cost), and does not remove validation.
Tags: D4 | S6 | Intermediate | extended-thinking-facts

### Q53. For high-volume, simple extraction, why is plain prompting + schema usually preferred over extended thinking?
A) Extended thinking returns malformed JSON on simple tasks
B) Extended thinking costs extra latency and tokens; reserve it for the hard slices, while simple high-volume extraction is cheaper and sufficient with plain prompting + schema
C) Extended thinking cannot be combined with a schema
D) Plain prompting guarantees schema conformance
Answer: B
Explanation: Extended thinking trades latency and tokens for reasoning depth, so it is wasteful on simple high-volume work where plain prompting plus a schema suffices. It does not categorically break JSON, can be combined with a schema, and plain prompting alone does not guarantee conformance.
Tags: D4 | S6 | Intermediate | extended-thinking-cost

## Section 13 — Streaming, Partial JSON, and Per-Slice Evaluation

### Q54. When streaming a structured/tool-use response, a developer runs a strict JSON parser on every chunk and treats exceptions as failures. Why is this a self-inflicted bug?
A) Streaming is not supported for tool use
B) Streamed JSON is invalid until the final token; you should accumulate the `input_json` deltas and parse once the block is complete (or use a tolerant partial parser), validating only after the structure is complete
C) The parser should run twice per chunk
D) Streaming disables schema validation entirely
Answer: B
Explanation: Partial JSON is invalid until complete, so per-chunk strict parsing will always throw mid-stream; accumulate deltas and parse/validate once the block finishes. Streaming is supported for tool use, double-parsing per chunk is nonsense, and validation is still required after completion.
Tags: D4/D5 | S6 | Advanced | streaming-partial-json

### Q55. A streamed structured output is cut off mid-object. What is the correct handling?
A) Best-effort parse whatever arrived and pad the missing fields with defaults
B) Detect incompleteness via the API signal (e.g. truncation / a length-limit `stop_reason`, missing closing braces) and route it into the retry/repair loop — do not guess missing fields
C) Treat the partial object as a successful extraction
D) Lower the temperature and accept the partial result
Answer: B
Explanation: A truncated object is a real failure; detect it via the API signal and retry/repair rather than padding or guessing. Best-effort parsing with defaults fabricates data, treating it as success is silent suppression, and temperature is irrelevant to truncation.
Tags: D4/D5 | S6 | Advanced | truncation-detection

### Q56. In S5/S6, an extraction batch reports 96% aggregate accuracy, yet one document type (handwritten forms) fails most of the time. What is the documented problem and fix?
A) No problem — 96% is above threshold, ship it
B) Aggregate accuracy masks a failing slice (anti-pattern #10); evaluate per document type / category and address the failing slice
C) Raise the iteration cap to push the aggregate higher
D) Filter out the handwritten forms and report success on the rest
Answer: B
Explanation: A high aggregate can hide a category that fails most of the time; the fix is per-slice evaluation (anti-pattern #10), not shipping on the headline number. Raising the cap does not fix the slice, and filtering out the failing class is exactly the blocking/filtering anti-pattern — fix the slice, do not hide it.
Tags: D4/D5 | S5/S6 | Advanced | per-slice-evaluation

### Q57. To make batch/CI extraction idempotent in S5/S6, what is the recommended persistence pattern?
A) Append a new row on every run, including retries
B) Derive a stable key from the source (content hash or business ID) and upsert on it, so a retried or replayed extraction overwrites cleanly without duplicates
C) Write half-validated output immediately to avoid losing work
D) Use a random UUID per run as the key
Answer: B
Explanation: Idempotent extraction uses a stable, content-derived key and upserts, so replays do not create duplicates; validate first, then commit. Appending on every run duplicates records, writing half-validated output invites duplicate retries, and a random per-run key defeats idempotency.
Tags: D4/D5 | S5/S6 | Advanced | idempotent-upsert

### Q58. Within a batch, one document repeatedly fails validation. What is the correct retry scope?
A) Reprocess the entire batch from scratch on any single failure
B) Scope the retry to the failed item using its stable key, so one bad document does not force reprocessing the whole batch
C) Drop the whole batch and report success
D) Lower the temperature for the entire batch and rerun all items
Answer: B
Explanation: Per-item retries keyed to the failing document isolate the failure and avoid reprocessing good items. Reprocessing the entire batch is wasteful, dropping the batch and reporting success is suppression, and a blanket temperature change rerunning everything is unnecessary and untargeted.
Tags: D4/D5 | S5/S6 | Advanced | per-item-retry-scope

## Section 14 — System/Role Prompts and Prompt Caching Overlap

### Q59. Where should the persona, domain, and standing constraints (e.g. "You are a senior tax analyst extracting fields from W-2 forms") live, and what additional benefit does that location provide?
A) In each user turn, which makes them easier to vary per request
B) In the `system` parameter, which both sharpens accuracy/tone and is the right home for prompt-cacheable stable context (long instructions, schemas, examples)
C) In the tool `input_schema`, replacing the system prompt
D) In a prefilled assistant turn
Answer: B
Explanation: The system parameter is the documented home for persona and standing constraints, and stable system content (instructions, schemas, examples) is a prime candidate for prompt caching. The task and data belong in the user turn; the schema and prefill serve different roles.
Tags: D4/D5 | S6 | Intermediate | system-prompt-caching

### Q60. When stable few-shot examples are placed in the system prompt for a high-volume extraction service, what efficiency benefit applies?
A) They become exempt from validation
B) They can be prompt-cached because they are stable across requests, reducing repeated processing cost on every call
C) They force strict structured outputs automatically
D) They lower the model's temperature
Answer: B
Explanation: Stable system-prompt content (including fixed examples) is cacheable, so the repeated stable prefix is not reprocessed from scratch each request — a real efficiency win for high-volume pipelines. Caching does not exempt output from validation, enable strict outputs, or change temperature.
Tags: D4/D5 | S6 | Intermediate | few-shot-cacheable

### Q61. The task and data for a given request should generally go in which turn, while persona and unchanging rules go in the system prompt?
A) The task and data go in the system prompt; persona goes in the user turn
B) The task and data go in the user turn; persona and standing constraints go in the system prompt
C) Both must go in a prefilled assistant turn
D) Both must go in the tool schema
Answer: B
Explanation: Anthropic recommends persona and standing constraints in the system parameter and the per-request task and data in the user turn — this split also maximizes prompt-cache reuse of the stable system content. The other options invert or misplace these.
Tags: D4/D5 | S6 | Foundational | system-vs-user-placement

## Section 15 — Integrated and Advanced Scenarios

### Q62. An S6 extraction prompt mixes the model's scratch reasoning into the same JSON object as the final structured answer, and downstream parsing breaks. What is the fix?
A) Tell the model to reason silently and hope it complies
B) Keep scratch reasoning in a separate `<thinking>` block (or the extended-thinking channel) so the JSON answer stays clean and parseable
C) Remove the schema so the reasoning has room
D) Parse the combined blob with a more lenient regex
Answer: B
Explanation: Reasoning must be isolated from the structured answer field; a separate `<thinking>` block (or extended thinking) keeps the JSON clean. "Reason silently and hope" is unreliable, removing the schema loses the contract, and lenient regex on a mixed blob is fragile.
Tags: D4 | S6 | Intermediate | reasoning-answer-separation

### Q63. A team claims "the schema is strict, so we can trust the model's extracted values without further checks." Why is this wrong on the exam?
A) Strict schemas frequently produce invalid JSON
B) Strict mode guarantees structural validity, not semantic correctness — a value can be schema-valid yet wrong (wrong date, wrong category), so semantic validation and an eval are still required
C) Strict schemas disable the retry loop
D) Strict schemas only apply to the first field
Answer: B
Explanation: "Trust because strict" confuses shape with meaning; structural validity does not imply the extracted value is correct, so semantic checks and evaluation remain necessary. Strict mode does not produce invalid JSON, disable retries, or apply to only one field.
Tags: D4 | S6 | Advanced | trust-strict-trap

### Q64. In S2, Claude is asked to implement a function and gets the algorithm wrong, but the output formatting (function signature, style) is fine. Which lever addresses the actual failure?
A) Add more few-shot examples of correctly-styled code
B) Use chain-of-thought or extended thinking to let the model reason through the algorithm before writing the code
C) Add an `enum` to constrain the output
D) Lower the temperature on the whole request
Answer: B
Explanation: The failure is reasoning, not format, so CoT/extended thinking is the lever; examples and enums fix format/shape, not the algorithm. Lowering temperature may help marginally but does not give the model room to reason through hard logic.
Tags: D4 | S2 | Intermediate | reasoning-vs-format-code

### Q65. Which end-to-end design best describes a production-grade S6 extraction pipeline?
A) Ask nicely for JSON, parse the prose, and catch exceptions by returning `{}`
B) Use a schema (strict structured outputs where supported, else tool `input_schema` + forced `tool_choice`), validate shape and semantics, feed specific errors back on a bounded corrective retry, escalate on exhaustion, upsert on a stable key, and evaluate per slice
C) Maximize the iteration cap and trust whatever parses
D) Increase temperature and model size until accuracy improves
Answer: B
Explanation: The production-grade pattern combines the strongest available output contract, validation of shape and semantics, specific corrective retries with a bounded cap and escalation, idempotent keyed upserts, and per-slice evaluation. Option A chains multiple anti-patterns (weak contract, prose parsing, silent suppression), and C/D are non-fixes.
Tags: D4 | S6 | Advanced | end-to-end-pipeline

### Q66. A prompt forbids the model from ever returning `null` or saying "not found," reasoning that "every field must always have a value." What is the predictable consequence?
A) Higher schema-conformance and fewer retries
B) Forced fabrication of values for genuinely-absent data — a hallucination cause the schema should instead handle with a nullable/sentinel path
C) Lower latency
D) Automatic enabling of strict structured outputs
Answer: B
Explanation: Removing the honest "absent" path forces the model to invent values, which is a documented hallucination cause; the fix is a nullable type or sentinel. It does not improve conformance, reduce latency, or enable strict outputs.
Tags: D4 | S6 | Intermediate | forbid-null-fabrication

### Q67. For a grounded Q&A over a large document set in S4, which prompt structure best supports accuracy and traceability?
A) Put the question first, omit document tags, and forbid quoting to keep answers short
B) Place documents near the top wrapped in tags with source/index metadata, put the query at the bottom, and ask the model to quote relevant passages before answering
C) Concatenate all documents with no tags and a high temperature
D) Split each document into its own request with no shared context
Answer: B
Explanation: Long-document grounding combines top-placed, tagged documents with metadata, bottom-placed query, and quote-first answering for traceability. Question-first, untagged concatenation, and per-document isolation all degrade long-context quality or grounding.
Tags: D4 | S4 | Advanced | grounded-qa-structure

### Q68. Which of these is the correct reading of the "cheapest effective rung first" principle in the documented prompt-engineering ladder?
A) Always start with extended thinking and work down
B) Reach for the cheapest technique that solves the problem (clear-and-direct, then examples, then CoT, etc.) before escalating to heavier methods
C) Always begin with strict structured outputs regardless of need
D) Skip clear-and-direct and go straight to multishot
Answer: B
Explanation: The ladder is ordered by impact and cost; you reach for the cheapest effective rung (clarity, examples, then CoT/extended thinking as needed) rather than jumping to heavy methods. Starting at the top wastes latency and tokens on problems that simpler rungs solve.
Tags: D4 | S2 | Intermediate | cheapest-effective-rung
