# Extended Thinking — CCA-F Reference

Extended thinking lets Claude perform explicit, multi-step internal reasoning before producing its final answer. When enabled, the model emits one or more `thinking` content blocks (its reasoning) ahead of the normal `text` answer block. This reference covers enabling thinking with a token budget, the content-block lifecycle, when to use it, interleaved thinking with tool use, redacted and summarized thinking, feature incompatibilities, prompt-caching interaction, and pricing. Source: extended thinking docs on platform.claude.com/docs and docs.anthropic.com.

## Enabling Extended Thinking with a Token Budget

Turn on extended thinking by adding a `thinking` object to the Messages request with `type: "enabled"` and a `budget_tokens` value. `budget_tokens` is the maximum number of tokens Claude may spend on internal reasoning before it must produce its answer.

The budget has two hard constraints the exam expects you to know. First, the minimum is **1024 tokens** — you cannot set a smaller thinking budget. Second, `budget_tokens` **must be less than `max_tokens`**, because thinking tokens are drawn from (and count toward) the overall output token allocation. If your reasoning budget equals or exceeds `max_tokens`, there is no room left for the visible answer and the request is invalid.

Treat `budget_tokens` as a target/ceiling, not a quota Claude must exhaust. The model spends what the problem needs up to the cap. For very large budgets, Anthropic recommends batch processing to avoid networking/timeout issues on long-running requests, and suggests starting at the minimum and increasing only if the task quality demands it. Larger budgets help on genuinely hard problems but add latency and cost.

**Exam signal (D4):** A distractor will set `budget_tokens` ≥ `max_tokens` or below 1024. The correct configuration always keeps `1024 ≤ budget_tokens < max_tokens`. Another distractor frames the budget as a guaranteed spend; it is a ceiling, not a floor.

## Thinking Content Blocks in the Response

When thinking is enabled, the response `content` array contains `thinking` block(s) **before** the final `text` block(s). A `thinking` block carries the reasoning text plus a cryptographic **`signature`** field. The signature lets the API verify that the thinking block was genuinely produced by Claude and was not altered when you send conversation history back in later turns.

Order matters: reasoning comes first, answer second. Your application should render or consume the `text` block as the user-facing answer, while the `thinking` block is diagnostic/auxiliary reasoning. Do not concatenate thinking into the answer blindly.

**Exam signal (D1/D4):** Recognize that the model returns *structured* reasoning blocks (with signatures), not a single merged string. Code that assumes only `text` blocks exist will mishandle thinking-enabled responses.

## When to Use Extended Thinking — and When Not To

Use extended thinking for tasks with genuine multi-step difficulty: complex math and proofs, intricate code generation or debugging, multi-constraint analysis, and **complex agentic planning** where the model must decompose a goal, sequence tool calls, and reason about trade-offs before acting. These are cases where the model's quality improves measurably when given room to reason.

Do **not** enable extended thinking for simple, latency-sensitive, or high-throughput tasks: straightforward lookups, short classifications, formatting, or any path where added latency and output-token cost are not justified by accuracy gains. Thinking adds tokens (cost) and wall-clock time; for trivial tasks it is pure overhead.

**Exam signal (D1/D4):** In a scenario, match thinking to task difficulty. Turning thinking on for a fast, simple classification step is a wrong answer (wasted cost/latency); turning it on for a hard agentic planning or multi-step reasoning step is correct. The decision is driven by reasoning complexity, not by output length or user sentiment.

## Interleaved Thinking with Tool Use (Beta)

Standard tool use lets the model think, then call a tool, but without interleaving the model reasons once up front. **Interleaved thinking** (a beta capability, enabled via a beta header) lets Claude think *between* tool calls — reason, call a tool, reflect on the tool result, reason again, call the next tool — producing more adaptive agentic behavior. This is especially valuable in multi-step agents where each tool result should inform the next decision.

### The Critical Rule: Pass Thinking Blocks Back Unmodified

This is the single most tested operational rule for extended thinking. **When tool use is involved, you must pass the `thinking` (and `redacted_thinking`) blocks back to the API unmodified in subsequent turns — including their `signature` fields.** When you return a `tool_result` to continue the conversation, the prior assistant turn's complete content (thinking blocks with intact signatures, plus the `tool_use` block) must be included exactly as received.

Why: the API uses the signature to verify reasoning continuity. If you strip, truncate, reorder, summarize, or edit the thinking blocks, or drop their signatures, the request fails validation (or the reasoning chain breaks). You may not synthesize or hand-author thinking blocks. The only correct handling is verbatim pass-through of what the API returned.

**Exam signal (D1/D2):** A scenario will show an agent loop that drops or rewrites thinking blocks before sending the next tool turn — that is the wrong answer. The right pattern echoes the assistant's prior thinking blocks (signatures intact) alongside the new `tool_result`. This pairs with the broader anti-pattern of mutating model-produced state in an agent loop.

## Redacted Thinking Blocks

Occasionally Claude's internal reasoning is flagged by safety systems and returned **encrypted** as a `redacted_thinking` block instead of plaintext `thinking`. This is expected and rare; it does not indicate that your prompt did anything wrong. The block's content is encrypted but still must be **passed back unmodified** in multi-turn / tool-use contexts, exactly like normal thinking blocks, so the model can continue its reasoning chain.

Your application should handle `redacted_thinking` gracefully — do not crash on an unrecognized block type, and do not attempt to decrypt or alter it. Render nothing user-facing for it if you cannot display it, but still preserve it in the message history you send back.

**Exam signal (D5):** Robust agent code tolerates `redacted_thinking` blocks (unknown-but-required content) and round-trips them intact. Code that errors out on a non-`text`/non-`thinking` block type is brittle and is the wrong answer.

## Summarized Thinking

For some models, the API returns a **summary** of the model's full thinking process rather than the complete raw reasoning. The model is billed for the **full** thinking tokens it generated, but the response surfaces a condensed summary of that reasoning. This keeps the reasoning useful for inspection while controlling response payload size, and helps prevent verbatim disclosure of the full chain of thought.

The practical consequence the exam targets: **output token counts in the response may not match the thinking tokens you are billed for**, because billing reflects the full reasoning while the visible summary is shorter. Do not build logic that assumes visible thinking length equals billed thinking tokens.

**Exam signal (D5):** If a scenario notes that thinking output "looks short" but the bill is higher, the explanation is summarized thinking — full reasoning is billed, a summary is returned. This is not a bug to "fix."

## Feature Incompatibilities and Constraints

Extended thinking changes what other parameters you may set. Know these for the exam:

- **`temperature` and related sampling constraints:** With extended thinking enabled you cannot freely tune sampling the way you normally would; thinking imposes constraints on `temperature` (and `top_p`/`top_k`). Attempting incompatible sampling settings alongside thinking is an error. Conceptually, the reasoning path is not meant to be heavily randomized by aggressive sampling overrides.

- **Cannot force `tool_choice`:** With extended thinking you cannot force tool use via `tool_choice` (e.g., requiring a specific tool or forcing *any* tool). Thinking is incompatible with forcing the model's hand on tool selection — the model must be free to reason about whether and which tool to call. Use `tool_choice: "auto"` semantics; do not force a tool.

- **Pre-filling the assistant turn** is not compatible with extended thinking (you cannot put words in Claude's mouth and also have it reason from scratch).

**Exam signal (D2/D4):** A distractor combines extended thinking with `tool_choice` forcing a specific tool, or with an assistant prefill, or with an out-of-range `temperature`. All are invalid. The correct agentic pattern lets the model decide tool usage while it reasons.

## Interaction with Prompt Caching

Extended thinking is compatible with prompt caching, but with important nuances. Thinking blocks from previous turns participate in the conversation context, and changes to thinking-related parameters or to the thinking blocks themselves can affect cache behavior. Notably, modifying or removing thinking content in the message history can invalidate cached prefixes, because the cached prefix must match exactly.

Practically: place stable, reusable content (system prompt, tool definitions, large context) behind cache breakpoints as usual, and avoid editing prior thinking blocks (which you must not edit anyway under the pass-back rule). Because thinking tokens are generated fresh each turn and count as output, they are not themselves "cached input" — caching saves on the repeated *input* prefix, not on regenerating reasoning.

**Exam signal (D5):** Combine the two rules: (1) never mutate prior thinking blocks, and (2) keep the cacheable prefix stable. A scenario that edits thinking history to "save tokens" both breaks signature validation and busts the cache — doubly wrong.

## Pricing — Thinking Tokens Billed as Output

Thinking tokens are billed as **output tokens** at the model's standard output rate. They are not free and not a separate cheaper tier. This means a generous `budget_tokens` directly increases cost, and (per the summarized-thinking note above) you are billed for the **full** thinking the model generated even when only a summary is returned.

Cost-control guidance: start near the minimum budget, raise it only when task accuracy demands it, reserve thinking for genuinely hard steps, and consider the Batch API for high-volume thinking workloads to manage long-running requests and cost.

**Exam signal (D4/D5):** If a question weighs accuracy vs. cost, remember thinking = output-priced tokens. The right answer enables thinking selectively on hard reasoning steps and keeps `budget_tokens` proportionate, rather than maxing the budget on every request.

## Quick-Reference Summary

- `thinking: { type: "enabled", budget_tokens: N }`, with **1024 ≤ N < max_tokens**.
- Response returns `thinking` (with `signature`) blocks **before** `text` answer blocks.
- Use for hard multi-step reasoning, math, complex code, **agentic planning**; skip for simple/low-latency tasks.
- Interleaved thinking (beta) reasons *between* tool calls.
- **Always pass `thinking`/`redacted_thinking` blocks back unmodified, signatures intact, in tool-use turns.**
- `redacted_thinking` = encrypted-but-required; round-trip it, don't crash on it.
- Summarized thinking: full reasoning billed, summary returned (visible length ≠ billed tokens).
- Incompatible with forced `tool_choice`, assistant prefill, and unconstrained `temperature`.
- Works with prompt caching, but never edit prior thinking (breaks signatures **and** cache).
- Thinking tokens are billed as **output**.
