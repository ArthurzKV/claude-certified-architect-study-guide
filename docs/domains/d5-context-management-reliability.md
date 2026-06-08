# Domain 5 — Context Management & Reliability (15%)

This domain tests whether you treat the context window as a finite, managed resource and whether your agents fail loud, recover gracefully, and prove correctness with per-slice metrics. The classic exam wrong answers here are "just make the window bigger," "trust the model to remember," and "swallow the error." Scenarios most often anchored to this domain are **S3 Multi-Agent Research System** (context isolation, error handling) and **S5 Claude Code for CI/CD** (batch reliability, retries, per-category eval).

## The CALM Framework: A Context Lifecycle You Can Reason About

Use CALM as a mental checklist for every long-running agent: **Curate** what enters the window, **Allocate** a token budget per component, **Limit** what persists turn-to-turn, and **Maintain** state externally so the window stays clean. CALM is a study mnemonic for the documented practice Anthropic calls **context engineering** — deliberately deciding what tokens occupy the window at each step, rather than dumping everything and hoping.

The lifecycle of context is: tools schema → system prompt → few-shot examples → retrieved/long context → running conversation. Each stage adds tokens that persist. The architect's job is to decide what is *stable* (cache it) versus *volatile* (compact or externalize it), and to never let irrelevant history accumulate.

**Exam signal:** When a scenario describes an agent that "degrades after many turns" or "starts ignoring earlier instructions," the right answer is a context-engineering intervention (compaction, isolation, external memory), NOT raising the iteration cap or switching to a bigger model.

## Context as a Finite Resource and Context Rot

A large context window is a budget, not free space. **Context rot** is the documented degradation in retrieval and instruction-following as the window fills with tokens — especially low-signal tokens like verbose tool outputs, stale turns, and redundant history. More tokens means more chances for the model to attend to the wrong thing.

The architectural consequence: every token you add has a cost in attention, latency, and dollars. Prefer the *smallest set of high-signal tokens* that lets the model act correctly. This is the "right altitude" principle — give enough context to decide, not a data dump.

**Exam signal:** A distractor will say "Claude supports a long context window, so include the entire 200-page manual every turn." The correct answer retrieves only the relevant sections (or summarizes), because flooding the window causes context rot and raises cost/latency.

## Prompt Caching: cache_control Breakpoints for Cost and Latency

Prompt caching lets Claude reuse a previously processed prefix instead of re-encoding it. You mark a cacheable prefix by attaching a `cache_control` breakpoint (type `"ephemeral"`) to the last block you want cached. Everything *before and including* that breakpoint becomes the cached prefix; a cache hit dramatically reduces input cost and time-to-first-token on repeated calls.

**Placement is the whole exam.** Cache after your *stable prefix*, ordered from most-stable to least-stable: **tools → system → few-shot examples → long static context**, with the dynamic, turn-by-turn user content *after* the last breakpoint. The prefix is matched as an exact token-for-token prefix, so any change before or at a breakpoint invalidates that cache and everything after it.

You can place **up to 4 cache breakpoints** in a request. Use multiple breakpoints to cache segments that change at different rates (e.g., one after the tool definitions, one after a large document) so a change to a later segment does not invalidate an earlier one.

**TTL:** the default cache lifetime is 5 minutes (refreshed on each hit), and a longer 1-hour TTL is available for content reused across a longer window. Choose 1-hour when a large stable document or tool set is hit repeatedly over a session but with gaps longer than 5 minutes; choose the default for tight loops.

**What invalidates the cache:** any edit to content at or before a breakpoint — changing a tool definition, editing the system prompt, reordering blocks, or even toggling settings that alter the serialized prefix (e.g., enabling/disabling extended thinking or changing tool definitions). Put *volatile* content (the latest user turn, retrieved snippets that change) AFTER the final breakpoint so it never breaks the cached prefix.

**Exam signal:** Given a multi-turn agent with a big system prompt and a fixed tool set, the correct optimization is "add a cache_control breakpoint after the tools/system prefix and keep dynamic input below it." A wrong answer puts the breakpoint after the latest user message (caches nothing reusable) or interleaves volatile data into the prefix (constant invalidation). Reference: platform.claude.com/docs — Prompt caching.

## Conversation Compaction: Summarize-and-Continue While Preserving State

Compaction replaces a long run of prior turns with a concise summary so the agent can keep going without exhausting the window. The right pattern: when the window approaches budget, generate a structured summary that **preserves key state** — decisions made, open tasks, established constraints, tool results that still matter, file paths/IDs — then continue from that summary plus the most recent turns.

The trap is lossy compaction. A good summary is *state-preserving*, not prose-pretty: it keeps the identifiers and commitments the agent needs to act, and drops only the conversational filler. In Claude Code, this is the `/compact` behavior; in the Agent SDK and Messages API you implement it yourself (or use built-in context/memory features where available).

**Exam signal:** S3's coordinator runs out of context mid-research. Correct: compact the history into a structured running summary (findings so far, sources cited, sub-questions remaining) and continue. Wrong: truncate the oldest messages blindly (drops the task definition), or restart from scratch (loses progress).

## Structured Note-Taking and External Memory

The most robust long-horizon pattern is to keep the *source of truth outside the window*. The agent writes structured notes — to a file, a scratchpad, a memory tool, or a task list — and reads back only what it needs for the current step. This keeps the window small and makes state durable across compaction, retries, and even crashes.

Use this for multi-step plans (a TODO file the agent updates), for research (a running findings document), and for extraction pipelines (write each validated record out immediately rather than holding all of them in context). External memory turns an unbounded conversation into a bounded working set plus durable storage.

**Exam signal:** "How should a multi-hour agent avoid losing progress when context is compacted?" Answer: persist state to external memory/notes and reload selectively — don't rely on the conversation history to hold it.

## Token Estimation and Budget Management

Architects must budget tokens the way they budget money. Roughly, English text is on the order of a few characters per token (do not memorize a magic ratio — Anthropic's official guidance is to count tokens, not guess), so for precise control use the **token counting** endpoint/utility to measure system prompt, tools, and inputs before sending. Reserve headroom for the model's output (`max_tokens`) and for tool-result growth across turns.

A practical budget: cap the static prefix (cache it), cap retrieved context per turn, and set a compaction threshold (e.g., compact when used context crosses a chosen fraction of the window). Account for *both* directions — extended thinking and tool results consume budget too, and tool outputs are often the fastest-growing, lowest-signal part of the window.

**Exam signal:** A distractor estimates tokens with a hard-coded "1 token = 4 chars" to gate a request. The better answer measures with the token-counting tool and reserves output headroom, because character ratios vary by content (code, JSON, non-English) and silently overflow.

## Sub-Agent Context Isolation: Keep Windows Clean

The headline reliability move in multi-agent systems is **context isolation**: give each sub-agent its own clean window scoped to one task, and return only a *compact result* to the coordinator — not the sub-agent's entire transcript. This prevents one agent's noisy tool calls and dead-ends from polluting the coordinator's reasoning, and it lets you parallelize without the windows colliding.

This is exactly the documented orchestrator–worker pattern from Anthropic's multi-agent research write-up: a lead agent decomposes the task, spawns subagents that each explore in isolation, and synthesizes their condensed outputs. Each subagent's exploratory context is *discarded*; only its findings cross the boundary. Reference: "How we built our multi-agent research system" (anthropic.com/engineering/built-multi-agent-research-system).

**Exam signal:** S3 asks how to stop the coordinator's window from filling up as subagents work. Correct: subagents run in isolated contexts and return summaries; the coordinator never ingests full subagent transcripts. Wrong: stream every subagent token back into the coordinator (defeats isolation, accelerates context rot).

## Multi-Turn Conversation Design

Design multi-turn flows so each turn carries forward only what the next step needs. Keep a stable, cacheable header (system + tools), append the new user/tool content below the cache breakpoint, and periodically compact the middle. Echo critical state (current goal, constraints, IDs) into a compact "state" block rather than relying on the model to re-derive it from a long history.

Make tool results lean: have tools return structured, minimal payloads (IDs and fields the agent needs) rather than raw dumps, since tool output is what fills windows fastest. When a result is large, store it externally and pass a reference/handle the agent can re-fetch on demand.

**Exam signal:** Questions contrast "include full history every turn" vs. "stable cached prefix + compacted middle + lean tool outputs." The latter is correct for cost, latency, and reliability.

## Reliability Patterns: Retries, Backoff, Idempotency, Graceful Degradation

Production agents must assume transient failure. Implement **retries with exponential backoff and jitter** for transient/rate-limit/5xx errors, and respect any retry guidance the API returns. Make retried operations **idempotent** — use idempotency keys or natural keys so a retried tool call (e.g., "create ticket") does not double-write. The Batch API is the right tool for large, latency-tolerant workloads (e.g., S5's many-file review): submit a batch, poll for completion, and reconcile results, instead of hammering synchronous calls.

**Graceful degradation:** when a dependency is down or a tool fails, degrade to a reduced-but-correct mode (e.g., return partial results clearly labeled, fall back to a cheaper path, or escalate) rather than crashing or fabricating. Always check the API **`stop_reason`** to know *why* a turn ended (`end_turn`, `tool_use`, `max_tokens`, `pause_turn`, etc.) and branch on it — never infer completion from the model's prose.

## Surface Errors, Never Suppress Them

The single most-tested reliability anti-pattern in this domain: **silently swallowing errors**. A tool that catches an exception and returns an empty result (or a generic "something went wrong") *as if it succeeded* blinds the model — it cannot retry, escalate, or adapt because it does not know anything failed. Return the diagnostic context (what failed, why, what to try next) in the tool result so the model can recover. This is the mirror image of "generic error messages that hide diagnostic context."

**Exam signal:** Given a tool that `except: return []`, the correct fix is to return a structured error with the failure reason (root-cause visible to the model), not to keep suppressing it. Hiding the failure is never the answer.

## Per-Category Evaluation Metrics (Not Just Aggregate Accuracy)

Reliability is *proven*, not assumed. An aggregate accuracy number can hide a category that is failing badly while the average looks fine. Evaluate **per slice**: per document type (S6 extraction), per ticket category (S1 support), per scenario, per tool. Track precision/recall on the slices that matter, and gate releases on the *worst* slice, not the mean.

This pairs with surfacing errors: per-category metrics only exist if failures are recorded rather than suppressed. In S5's CI/CD review pipeline, report pass/fail and accuracy *per check type and per file class* so a regression in one category is visible immediately.

**Exam signal:** A distractor reports "94% overall accuracy, ship it." The correct answer breaks accuracy down per document type / category and finds the masked failing slice before shipping.

## Top Traps (Memorize the Wrong Answers)

- **Bigger window / bigger model as the fix** for degradation — the fix is context engineering (compaction, isolation, external memory), not more tokens.
- **Dumping full documents/history every turn** — causes context rot and cost; retrieve/summarize instead.
- **cache_control after the volatile content** (or interleaving dynamic data into the prefix) — caches nothing reusable / constant invalidation. Cache the stable prefix; keep dynamic content below the last breakpoint.
- **Blind truncation of oldest messages** — drops the task definition; use state-preserving compaction.
- **Streaming full subagent transcripts to the coordinator** — defeats context isolation; return compact results only.
- **Hard-coded char→token ratios to gate requests** — measure with token counting; reserve output headroom.
- **`except: return []` / generic error strings** — surface the diagnostic so the model can recover; never suppress.
- **Inferring completion from prose ("I'm done")** — branch on `stop_reason`.
- **No idempotency on retried writes** — double-creates resources; use idempotency/natural keys.
- **Aggregate-only metrics** — mask per-slice failures; evaluate per category/document-type.

## Quick Mapping to Scenarios

- **S3 Multi-Agent Research:** subagent context isolation, coordinator compaction, structured findings as external memory, graceful degradation when a subagent fails (surface, don't suppress).
- **S5 CI/CD:** Batch API for many-file review, retries/backoff/idempotency, multi-pass review with fresh context, per-check-type and per-file-class eval metrics, `stop_reason`-driven control flow.
- **S6 Extraction (cross-cutting):** write validated records to external memory immediately; per-document-type accuracy; validation-retry surfaces the schema error rather than hiding it.
