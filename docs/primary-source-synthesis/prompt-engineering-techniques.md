# Prompt Engineering Techniques (CCA-F: Domain D4 — Prompt Engineering & Structured Output)

This reference distills Anthropic's documented prompt-engineering techniques into exam-ready rules. Each section gives the RIGHT pattern, WHY it works, the common trap, and an "Exam signal:" note tying the concept to scenario questions — especially **D4 (Prompt Engineering & Structured Output, 20%)** and **S6 (Structured Data Extraction)**. Source of record: the prompt engineering guide at platform.claude.com/docs (Build with Claude → Prompt engineering) and docs.anthropic.com.

A useful framing from the docs: apply techniques roughly in order of impact — be clear and direct first, then add examples, then chain-of-thought, then XML structure, then system prompts/prefill, then prompt chaining. Reach for heavier machinery (chaining, agents) only after the cheap, high-leverage techniques are exhausted.

## Be clear and direct: state exactly what you want and give the motivation

Rule: Tell Claude precisely what to do, in what order, and in what format — as you would brief a new, capable contractor who lacks your context. Include the *why* (the goal/motivation), because Claude generalizes better when it understands intent rather than just following a literal rule.

Why: Most "Claude got it wrong" cases are underspecified prompts, not model failure. Explicit instructions plus motivation let the model resolve ambiguity the way you intended and apply your rule to edge cases you didn't enumerate.

Example: Instead of "Summarize this," write: "Summarize the support ticket below in 3 bullets for an on-call engineer who must decide whether to page the database team. Lead with the customer impact." The audience + decision context is the motivation.

Trap: Burying the actual instruction in vague politeness, or relying on the prompt to *enforce* a hard business rule. Critical, deterministic rules (refund limits, PII redaction) belong in code/hooks, not in pleading prose — this is anti-pattern #3 (prompt-based enforcement of critical rules).

Exam signal (D4): The "best prompt" option adds explicit format + audience + motivation. Distractors are terse and contextless, or try to enforce a safety/business invariant purely through wording.

## Use examples (multishot prompting) and how many

Rule: Show 3–5 diverse, relevant examples of the exact input→output mapping you want. Examples are the single most effective lever for steering format and handling edge cases — far more reliable than describing the format in prose.

Why: Claude pattern-matches on demonstrations. Multishot reduces format drift, encodes tone, and teaches the handling of tricky cases (nulls, ambiguous inputs) by example rather than by rule.

Make examples count: cover edge cases, not just the happy path; keep them *diverse* so Claude doesn't overfit one phrasing; wrap them in tags (e.g. `<examples>` / `<example>`) so they're clearly delimited from the live input.

Trap: One example (the model overfits its surface form), or examples that all look alike (no edge-case coverage). For extraction (S6), failing to include an example that shows the desired behavior for a *missing* field invites hallucinated values.

Exam signal (D4/S6): When extraction output is inconsistent, the fix is "add diverse few-shot examples including null/edge cases," not "raise temperature" or "add a longer prose description."

## Let Claude think: chain of thought, but structured

Rule: For analysis, math, multi-step decisions, or anything requiring reasoning before an answer, instruct Claude to think step by step *before* producing the final answer — and give the thinking a structured home, e.g. a `<thinking>` section followed by an `<answer>` section.

Why: Reasoning that precedes the answer improves accuracy on complex tasks. Structuring it (separate tags) keeps the scratch work out of the final deliverable and makes the answer trivially parseable.

Example: "First, in `<thinking>` tags, list the eligibility criteria and check each against the account. Then give the verdict in `<decision>` tags as exactly `APPROVE` or `DENY`."

Trap: "Think step by step" with no structure, so reasoning bleeds into the output you need to parse. Note: with **extended thinking**, you do not prefill or hand-author the thinking block — you enable the thinking budget and let the model reason; don't also force a competing `<thinking>` scaffold. Plain CoT (tags in the prompt) is for non-thinking calls.

Exam signal (D4): Correct answer separates reasoning from the parseable result. A distractor asks for "just the answer" on a task that clearly needs reasoning, hurting accuracy.

## Use XML tags to delimit inputs, instructions, and sections

Rule: Wrap distinct parts of the prompt — documents, data, instructions, examples, output spec — in clearly named XML-style tags (`<document>`, `<instructions>`, `<examples>`, `<format>`). Reference those tag names in your instructions.

Why: Tags remove ambiguity about where the data ends and the instruction begins, prevent the model from confusing your instructions with quoted content, and make multi-part prompts robust. They also pair naturally with prefill and parsing (you can tell Claude to put the result in a specific tag).

Example: "Using only the facts in `<transcript>...</transcript>`, answer the question in `<question>...</question>`. Put your answer in `<answer>` tags."

Trap: Pasting a long document inline with the question and no delimiters — the model may follow instructions embedded in the document, or blur data and task (a prompt-injection and accuracy risk).

Exam signal (D4/S6): The robust prompt tags inputs and names the output tag; weaker options concatenate everything as free text.

## System prompts and role assignment

Rule: Put durable role, persona, expertise, and global rules in the **system prompt**; put the specific task and data in the user turn. Assigning a role ("You are a senior SRE triaging incidents") sharpens tone, vocabulary, and judgment.

Why: A clear role primes domain-appropriate behavior and is a clean separation: system = who Claude is and standing constraints; user = what to do right now. This is also where you encode standing output conventions.

Trap: Stuffing the task-specific input into the system prompt, or relying on the system prompt alone to enforce hard guarantees (still anti-pattern #3 — back invariants with code). Role assignment improves *style and framing*; it is not a security boundary.

Exam signal (D3/D4): In Claude Code / Agent SDK contexts, the system prompt (and `CLAUDE.md` as standing context) carries persistent rules; per-task specifics go in the message.

## Prefill the assistant response to steer format

Rule: Start Claude's reply for it by providing the opening of the assistant turn. Prefilling with `{` forces JSON and skips preambles; prefilling with `<answer>` forces output straight into that tag; prefilling a first label enforces a structure.

Why: Prefill is the most deterministic non-tool way to lock output format and suppress chatty preambles ("Sure, here's…"). It guarantees the response *begins* exactly where you want parsing to start.

Trap: Prefilling and then expecting Markdown fences or prose too. Also: with **extended thinking enabled you cannot prefill the assistant turn** — these features conflict. For strict schemas, prefer **tool use / structured outputs** over prefill alone (see S6).

Exam signal (D4/S6): "How do you stop Claude from adding explanatory text before JSON?" → prefill with `{` (or, better for guaranteed schema, use a tool/structured output). A distractor says "ask it nicely in the prompt."

## Chain complex prompts into discrete steps

Rule: Decompose a complex task into a sequence of focused prompts, passing each step's output (ideally tagged) into the next: e.g. extract → validate → summarize. Each call does one job well.

Why: Single mega-prompts juggling many subtasks lose accuracy and are hard to debug. Chaining isolates failures, lets you inspect intermediate artifacts, and allows a fresh context for steps that benefit from one — e.g. a *separate* review pass that doesn't inherit the author's reasoning (avoiding anti-pattern #9, same-session self-review).

Trap: Cramming extract + transform + judge + format into one prompt and then being unable to tell which subtask failed. The opposite over-correction — chaining trivial steps that one clear prompt handles — adds latency for no gain.

Exam signal (D1/D5/S5): Multi-pass CI review and S3 coordinator/subagent designs favor chained/separated steps; the self-review-in-same-context option is the trap.

## Long-context tips: documents near the top, query at the end, quote first

Rule: For long inputs, place the large documents/data **near the top** of the prompt and put the actual question/instruction **at the end**. For grounded Q&A over long docs, ask Claude to **first quote the relevant passages** (in tags) and only then answer using those quotes.

Why: With long contexts, trailing instructions are attended to reliably, and "quote-then-answer" forces the model to retrieve evidence before reasoning — cutting hallucination and making answers auditable. Tag and (optionally) index multiple documents so Claude can cite which one it used.

Example: `<documents>`…long material…`</documents>` then `<question>`…`</question>`, with "Quote the sentences supporting your answer in `<quotes>` tags before answering."

Trap: Leading with the question and trailing a 50-page appendix, or asking for an answer with no grounding step over long material. (Conceptually, prompt caching pairs well here — a stable long prefix can be cached — but caching is a cost/latency optimization, not a correctness technique.)

Exam signal (D4/D5): Long-doc questions reward "docs first, query last, quote relevant parts first." Distractors invert the order or skip grounding.

## Reduce hallucinations: allow "I don't know," ground, and cite

Rule: Explicitly give Claude an out — permit "I don't know" / "not stated in the source" when the answer isn't supported. Constrain answers to provided context ("answer only from `<context>`"), and require citations/quotes back to the source.

Why: Without an explicit out, models fill gaps with plausible fabrication. Permission to abstain plus grounding plus citation turns a guessing task into a retrieval task, and citations make wrong answers detectable.

Trap: Demanding a confident answer for every question; trusting a model-emitted **confidence score** to gate behavior (anti-pattern #4). Confidence numbers are not calibrated — use grounding/verification, not self-reported certainty.

Exam signal (D4/D5/S6): The hallucination-reduction answer is "allow abstention + ground in provided context + require citations." Routing on a self-reported confidence field is the trap.

## Best practices for the latest Claude models

Rule: With current Claude models, be **explicit** about desired behavior rather than relying on implicit defaults, add the **motivation** behind each instruction, and where multiple independent actions are needed, **encourage parallel tool calls** ("call all relevant tools simultaneously rather than one at a time").

Why: Newer models follow precise instructions faithfully and act on stated intent, so spelling out the target behavior (and why) yields more reliable results than terse commands. Explicitly inviting parallel/concurrent tool calls improves agent latency and throughput when subtasks are independent.

Also: prefer positive, concrete instructions ("do X in format Y") over long lists of prohibitions; reserve hard constraints for code/hooks; and steer format with tool use, structured outputs, or prefill rather than hoping for it.

Trap: Assuming the model will infer unstated preferences; or serializing independent tool calls when they could run in parallel. Over-reliance on negative "do NOT" lists tends to be less effective than stating the desired behavior.

Exam signal (D1/D2/D4): Latest-model best practice = explicit + motivated instructions + parallel tool calls for independent work. Distractors say "keep the prompt minimal and let the model figure it out," or force sequential tool calls.

## Structured output for extraction (D4 ↔ S6 bridge)

Rule: For machine-consumed output, don't parse free text — define the shape and let the model fill it. Use **tool use** with an input JSON Schema (the model's `tool_use` arguments are the structured result), or **structured outputs** to constrain to a schema. Validate the result and run a **validation-retry**: on schema failure, feed the validation error back so the model can correct it.

Why: Tool/structured outputs give schema-conformant results far more reliably than "respond in JSON" prose plus regex. Feeding the *actual* validation error back (not a generic message) is anti-pattern #6's fix — diagnostic context lets the model self-correct.

Trap: Prose JSON requests with brittle string parsing; silently dropping records that fail validation (anti-pattern #7 — suppressing errors and returning empties as if successful); and reporting one aggregate accuracy number that hides per-document-type failures (anti-pattern #10 — evaluate per slice).

Exam signal (S6): Robust extraction = JSON Schema via tool_use/structured output + validate + retry-with-error-context + per-category evaluation. The traps are regex-on-prose, silent suppression, and a single masked accuracy metric.

## Quick technique-selection checklist

Start clear and direct (state the task, format, audience, and motivation). Add multishot examples (3–5, diverse, edge cases). Add structured chain-of-thought when reasoning is needed. Use XML tags to delimit every part. Set role/standing rules in the system prompt. Prefill or — better for schemas — use tool use / structured outputs to lock format. Chain into discrete, separately-contextualized steps for complex or review tasks. For long inputs: documents first, query last, quote-then-answer. To cut hallucination: allow "I don't know," ground in provided context, require citations. Enforce hard rules in code/hooks, never in the prompt alone.

Sources: platform.claude.com/docs (Prompt engineering overview, Be clear and direct, Multishot prompting, Chain of thought, Use XML tags, System prompts, Prefill Claude's response, Chain prompts, Long context tips, Reduce hallucinations, Claude best practices); docs.anthropic.com/en/docs/build-with-claude.
