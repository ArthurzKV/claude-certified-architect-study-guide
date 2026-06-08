# Prompt Caching on the Claude API

Source: Anthropic Docs — "Prompt caching" (https://platform.claude.com/docs/build-with-claude/prompt-caching). Maps to **D5 Context Management & Reliability (15%)** and surfaces heavily in **S5 Claude Code for CI/CD** (repeated multi-pass review of the same large input).

## What Prompt Caching Is and Why It Exists

Prompt caching lets you mark a **prefix** of a request so Anthropic stores the processed internal state of those tokens on its servers. On a subsequent request that begins with the *exact same* prefix, Claude reuses the cached state instead of re-processing those tokens from scratch. The result is **lower cost** (cached reads are far cheaper than processing fresh input) and **lower latency / faster time-to-first-token**, because the model skips the expensive prefill work for the cached portion.

The win is largest when you repeatedly send the same stable content across many requests: a long system prompt, a big set of tool definitions, a large document or codebase being analyzed, or few-shot examples. Anything that does not change between calls is a caching candidate.

**Exam signal:** Questions frame caching as a cost/latency optimization for *repeated prefixes*. If a scenario sends the same 30-page contract or the same large codebase context on every iteration of a loop or every reviewer pass, the correct answer caches that stable prefix once and varies only the suffix (the per-request question/diff).

## How You Turn It On: `cache_control` Breakpoints

Caching is opt-in. You add a `cache_control` object to the *last* content block you want included in a cached prefix. The marker has the shape `{"type": "ephemeral"}`. Everything in the request **up to and including** that block becomes a cacheable prefix.

```json
{
  "type": "text",
  "text": "<large stable system content>",
  "cache_control": {"type": "ephemeral"}
}
```

The block carrying `cache_control` is a **breakpoint** — it defines where the cached prefix ends. You do not annotate every block; you annotate the boundary.

**Exam signal:** The keyword is `cache_control` with `type: "ephemeral"`. Distractors invent a top-level `cache: true` request flag or a `cache_ttl` header — those are wrong; caching is expressed per content block via `cache_control`.

## Up to Four Breakpoints

A single request may define **up to 4** `cache_control` breakpoints. Multiple breakpoints let you cache nested layers that change at different rates — for example, one breakpoint after tool definitions, another after the system prompt, another after a large pinned document. The system also looks *backward* from each breakpoint to find the longest previously-cached prefix, so even with one breakpoint you can get a partial cache hit on the unchanged earlier portion.

Use extra breakpoints when different segments have different stability/churn. If the whole prefix is one monolithic stable block, a single breakpoint at its end is enough.

## Minimum Cacheable Prefix Length

A prefix must be long enough to be worth caching. The documented minimum is approximately **1024 tokens** for most current Claude models, and approximately **2048 tokens** for the smallest/fastest models (Haiku-class). If the content before your breakpoint is shorter than that minimum, the request still succeeds but **nothing is cached** — the breakpoint is silently ignored and you pay normal input pricing.

**Exam signal:** A scenario that caches a tiny ~200-token system prompt and reports "no cost savings" is hitting the minimum-length floor. The fix is not more breakpoints — it is recognizing the prefix is below the cacheable minimum. Treat ~1024 / ~2048 as model-dependent thresholds; cite them as approximate and check the model's docs for the exact figure.

## Time-To-Live: 5-Minute Default vs 1-Hour Optional

A cache entry has a **TTL** (lifetime). The **default TTL is about 5 minutes**, and it is a *sliding* window — each cache hit refreshes the clock, so a frequently-reused prefix stays warm indefinitely as long as calls keep arriving within the window. If no request reuses the prefix before it expires, the entry is evicted and the next call is a cache miss (a fresh write).

You can optionally request a longer **1-hour TTL** for prefixes you reuse less frequently but still want to persist across longer gaps (e.g., a batch job that revisits the same context every several minutes). The 1-hour option is selected through the `cache_control` configuration (the `ttl` setting) and costs more to write than the 5-minute tier.

**Exam signal:** If iterations of a workflow are spaced more than ~5 minutes apart and the scenario complains about repeated cache misses, the answer is the **1-hour TTL**, not "increase iteration cap" or "shorten the prompt." Conversely, for a tight loop firing every few seconds, the 5-minute default already keeps the entry warm.

## Pricing Shape: Writes Cost More, Reads Cost Much Less

Caching shifts cost, it does not make it free. Three rates apply (multipliers are **approximate** and tier/model-dependent — describe the *shape*, verify exact numbers in pricing docs):

- **Cache write** (first time a prefix is stored): *more* than base input — roughly **~1.25×** base for the 5-minute tier and **~2×** base for the 1-hour tier. You pay this premium once per write.
- **Cache read** (a hit on an existing prefix): *far cheaper* than base — on the order of **~0.1×** base input (a ~90% discount on the cached tokens).
- **Uncached input** (the varying suffix and any cache miss): normal base input pricing.

The economics: you pay a one-time write premium, then every subsequent hit is ~10% of base for those tokens. Caching is a **net win when the prefix is reused enough times** that accumulated read savings exceed the write premium. For a prefix sent once and never reused, caching is *more* expensive than not caching.

**Exam signal:** Expect a cost-reasoning question. "We cache but it costs more" usually means the prefix is rarely reused (write premium with no read amortization) or it expired between calls (write, miss, write again). The right framing is reuse frequency vs. TTL, not "disable caching." Mark the multipliers as approximate; the load-bearing facts are: writes > base, reads << base.

## Required Ordering: Tools → System → Messages

The cacheable prefix follows the **fixed request order** in which Claude assembles content:

1. **`tools`** (tool definitions) — first.
2. **`system`** (system prompt) — next.
3. **`messages`** (conversation turns) — last.

A cached prefix is always a contiguous span from the very start of this sequence up to a breakpoint. You **cannot** cache something in the middle while leaving earlier content uncached — *everything before the breakpoint is part of the cached prefix*. So a breakpoint placed in the `system` block caches all tools plus the system content up to that point; a breakpoint in `messages` caches tools, system, and the earlier messages.

**Exam signal:** A distractor will propose caching only the system prompt while putting *changing* tool definitions before it. That is impossible — tools come first in the prefix, so if tools change, the system cache is invalidated too. Stable content must be ordered *earliest*.

## What Invalidates the Cache

A cache hit requires the prefix to match **exactly**, byte-for-byte, from the beginning. Any change at or before a breakpoint invalidates that cache entry (and everything after it), forcing a new write. Common invalidators:

- **Editing tool definitions** — adding, removing, renaming, or re-describing any tool. Because `tools` is first in the prefix, *any* tool change busts every downstream cache layer.
- **Changing the system prompt** (even whitespace or a single character within the cached span).
- **Toggling or changing extended-thinking settings**, which alters the request prefix.
- **Modifying any earlier message** in a cached `messages` prefix.
- **TTL expiry** — letting the window lapse evicts the entry; the next call is a miss/write.

Changes *after* the last breakpoint (your varying suffix) do **not** invalidate the cache — that is exactly the point. Keep the churning content at the end.

**Exam signal:** "Why is our cache hit rate near zero?" The classic root cause is mutating something inside the cached prefix every request — e.g., injecting a timestamp or a per-request ID into the system prompt, or shuffling tool order. Fix the *source*: move the volatile token out of the prefix into the suffix; do not "work around" it by disabling caching.

## Caching With Tools and With Extended Thinking

**Tools:** Large tool schemas are excellent cache candidates because they are stable and sit first in the prefix. Put a breakpoint after the tool block to cache all definitions once and reuse them across the session. But remember the flip side — any tool edit invalidates everything, so finalize your tool set before relying on tool caching.

**Extended thinking:** Caching is compatible with extended thinking. Note that thinking parameters are part of the request configuration; changing them affects the prefix. Within a turn, the cached prefix (tools/system/large context) is reused while the model thinks and produces the answer over the varying suffix. Treat thinking config as "stable content" you set up front so it does not bust the cache mid-workflow.

## Best Practices: Stable First, Breakpoint After

Order content from **most stable to most volatile**, then drop the breakpoint at the boundary between them:

1. **Tool definitions** (stable across the session) — first.
2. **System prompt** (role, rules, persistent instructions) — next.
3. **Large shared context** (the document, the codebase snapshot, the knowledge base) — next.
4. **Few-shot examples** (fixed demonstrations) — next.
5. *Breakpoint here* (`cache_control: {type: "ephemeral"}`) — after all of the above.
6. **The varying suffix** (the user's actual question, the current diff, the per-item input) — after the breakpoint, uncached.

This maximizes the cached prefix and minimizes the uncached suffix, which is what drives the cost/latency win. Keep anything that changes per request strictly *after* the breakpoint.

**Exam signal (S5 / D5):** A CI/CD multi-pass review sends the same large diff or codebase context to several reviewer passes. The architecturally correct design caches that shared context once (stable prefix + breakpoint) and varies only each pass's instruction/question. Watch for the dual trap: caching is *not* a substitute for using a **fresh context per reviewer** to avoid self-review bias (anti-pattern #9) — you cache the shared *input*, but each independent reviewer still reasons separately. Caching optimizes cost; it does not collapse the reviewers into one biased session.

## Quick Reference Checklist

- Mark the boundary with `cache_control: {"type": "ephemeral"}` on the last block of the prefix.
- Up to **4** breakpoints per request.
- Prefix must clear the minimum (~**1024** tokens; ~**2048** for the smallest models) or nothing caches.
- TTL: **~5 min** sliding default, optional **~1 hr** tier (higher write cost).
- Pricing shape: **write > base** (~1.25× / ~2×), **read << base** (~0.1×). Net win requires enough reuse.
- Order: **tools → system → messages**; everything before the breakpoint is the cached prefix.
- Any change at/before a breakpoint (tools, system, thinking config, earlier messages) invalidates the cache. Keep volatile tokens in the suffix.
