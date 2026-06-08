# CALM Framework & Context Lifecycle (CCA-F D5)

Pure reference for **Domain 5 — Context Management & Reliability (15%)**. CALM is the exam's mental model for reasoning about a finite context window across a long-running agent or workflow. Treat the context window as a managed resource with a *budget*, a *lifecycle*, and *durability concerns* — not as infinite scratch space. Grounded in publicly-documented Anthropic behavior: prompt caching (https://docs.claude.com/en/docs/build-with-claude/prompt-caching), context windows and token counting (https://docs.claude.com/en/docs/build-with-claude/context-windows), the Agent SDK / context editing & memory tooling (https://docs.claude.com/en/docs and the Claude Agent SDK), the multi-agent research write-up (https://www.anthropic.com/engineering/built-multi-agent-research-system), and "Building Effective Agents" (https://www.anthropic.com/research/building-effective-agents).

## CALM Defined — the Context-Lifecycle Mental Model

**C — Context awareness.** Know what is actually in the window right now: the system prompt, tool definitions, conversation history, tool results, retrieved documents, and any thinking blocks. Know the *total* window size for the model in use and how close you are to it. You cannot budget what you don't measure.

**A — Allocation / budgeting.** Deliberately divide the window into named buckets (system + tools, static context, working history, headroom for output and thinking). Each bucket has a cap. When a bucket would overflow, you compact or offload — you never just let history grow until the model truncates or errors.

**L — Lifecycle management.** Plan how content *enters, ages, and exits* the window. Tool results that mattered three turns ago are dead weight now. Lifecycle includes **compaction** (summarize-and-replace), context editing (clearing stale tool results), and pruning, all triggered by thresholds rather than left to chance.

**M — Memory / notes & retrieval.** Durable state lives *outside* the window — files, a scratchpad, a memory store, a database — and is *retrieved on demand*. The window holds a pointer or a summary; the full artifact is fetched only when needed. This is what lets an agent run far longer than a single window.

**Exam signal:** any stem about "the agent loses track / repeats work / forgets the original goal on long tasks" is a CALM failure. The right answer almost always involves **external memory + compaction + retrieval**, not "use a bigger model" or "raise the iteration cap."

## Token-Budget Worksheet (the Allocation step)

Budget *before* you run, top-down. A workable worksheet:

1. **Window total** — the model's full context limit (token-count it; don't estimate by characters). Use the count-tokens endpoint or SDK helper rather than guessing.
2. **Reserve output headroom** — subtract `max_tokens` for the model's reply. Output competes with input for the same window.
3. **Reserve thinking headroom** — if extended thinking is on, reserve its budget too; thinking tokens consume the window.
4. **Fixed overhead** — system prompt + tool definitions. These are large and *constant*, which is exactly why they go first in the cache (see below).
5. **Static context** — few-shot examples, retrieved reference docs, the CLAUDE.md / project context. Cap this; rank by relevance and trim.
6. **Working budget** = whatever remains. This is your conversation history + live tool results. When it nears its cap, **compact**.

Rule of thumb: never plan to fill the window. Leave headroom — quality and instruction-following degrade as you approach the limit (context rot, below), and you need room for the final answer.

**Exam signal:** distractor "just send everything and let the model sort it out." Wrong — that invites truncation, rot, and cost. The right pattern is *measure, cap each bucket, offload the overflow.*

## cache_control Breakpoint Placement Playbook

Prompt caching reuses an exact, **prefix** of the request. A cache hit requires that everything **before** the breakpoint be byte-for-byte identical to a prior request. Therefore order content **most-static first, most-dynamic last**, and place breakpoints at stability boundaries.

**Canonical order (stable → volatile):**

1. **`tools`** — tool definitions change least often. Cache them first.
2. **`system`** — system prompt / instructions. Stable across a session.
3. **Few-shot examples & large static context** — reference docs, schemas, CLAUDE.md-style context. Stable for the task.
4. **Dynamic messages** — the live conversation and fresh tool results. These change every turn; they go *last* and typically stay *outside* the cached prefix.

**Breakpoint mechanics to memorize:**

- You may set **up to 4** `cache_control` breakpoints. The system also automatically checks for hits at earlier breakpoints, so placing one at the end of the largest stable block is the high-value move.
- A breakpoint caches **everything from the start of the prompt up to and including that block** (a cumulative prefix), not just the marked block in isolation.
- Cache writes cost more than normal input tokens; **cache reads are much cheaper**. Caching pays off when a stable prefix is reused across many requests (multi-turn sessions, batch over a shared system prompt, agent loops).
- **TTL:** default **5-minute** refreshing window; an extended **1-hour** TTL is available. Every cache *hit* refreshes the entry's TTL, so an active session keeps the cache warm. The 1-hour option suits sessions with gaps longer than five minutes between calls.

**What invalidates the cache (forces a re-write of everything from that point onward):**

- **Any change to content *before* the breakpoint** — even one token. Because the cache is a *prefix* match, editing the system prompt invalidates the few-shot and message caches downstream of it.
- **Reordering** tools, system, or examples.
- **Toggling features that alter the prefix** — e.g. turning extended thinking on/off, or changing tool definitions, can change the cacheable prefix.
- **TTL expiry** without a refreshing hit.

**Exam signal (S5 batch / long sessions):** the right caching answer puts the big invariant block (system + tools + schema/examples) in the cached prefix and appends only the per-item dynamic input. The trap is interleaving dynamic content *before* the static block (e.g. putting a per-request timestamp or user ID at the top), which destroys every downstream cache hit. Another trap: caching content that's used only once — you pay the write premium with no reuse.

## Compaction — Triggers & Preserving Critical State (the Lifecycle step)

**Compaction** = replace a span of verbose history with a concise summary so the loop can continue without losing the thread. It is the primary mechanism that lets agents exceed a single window.

**Trigger on a threshold, not on vibes.** Common triggers: working budget crosses a percentage of the window (e.g. approaching the cap with headroom still reserved), a turn count, or a tool-result volume threshold. The Agent SDK supports automatic compaction/context-editing as the window fills; you set the policy, the runtime enforces it.

**What MUST survive a compaction (preserve explicitly):**

- The **original objective / task spec** — the single thing agents most often lose.
- **Decisions made and their rationale**, and any commitments or constraints discovered.
- **Open sub-tasks / the plan state** — what's done, what's pending.
- **Key identifiers and facts** needed downstream (IDs, file paths, schema, the validated values so far).
- **Unresolved errors** still being worked.

**What is safe to drop:** verbose raw tool output already summarized, superseded intermediate reasoning, exploratory dead-ends, and stale retrieved documents.

**Exam signal:** "long agent run eventually drifts off-task / re-does completed steps." Cause = naive truncation that dropped the goal and plan state. Fix = **summary-based compaction that pins objective + plan + key facts**, plus writing those to external memory so they can be re-read. Trap distractors: "increase max iterations," "use a larger window model" (delays the problem; doesn't manage lifecycle), or "drop the oldest N messages" (blind truncation can discard the goal).

## Structured Note-Taking & External Memory Patterns (the Memory step)

The durable-state pattern: the agent **writes structured notes to an external store** (a file, scratchpad, memory tool, or DB) and keeps only a compact reference in-window. On later turns it **retrieves** the relevant note rather than carrying the full text the whole time. This is the mechanism behind agents that run for hundreds of steps.

**Good note structure:** stable, parseable, and slice-able — e.g. a running plan with checkboxes, a "findings" log keyed by source, a "decisions" record, and a "state" block (current objective + next action). Structured notes (Markdown/JSON sections) survive compaction better than prose because the important fields are explicit and re-loadable.

**Retrieval over recall:** keep large reference material (codebase, knowledge base, prior research) *outside* the window and pull only the relevant slice on demand (search/RAG, file reads, MCP resources). The window holds the *query result*, not the corpus.

**Exam signal:** "how does the agent remember across compaction / across a fresh subagent?" → it persisted structured state to external memory and re-reads it. The trap is relying on the conversation history alone to carry long-term state — history is volatile and gets compacted away.

## Subagent Context Isolation

In hub-and-spoke / orchestrator-workers designs, each **subagent runs in its own fresh context window**. The coordinator delegates a scoped task; the subagent does the noisy exploration in *its* window and returns a **condensed result**, not its full transcript. This is a context-management win, not just an orchestration one: the coordinator's window stays clean and small, and parallel subagents multiply *effective* context without bloating any single window.

**Isolation rules to remember:**

- Subagents get a **clear, self-contained brief** (objective, scope, output format) because they don't share the coordinator's history.
- They return **compressed findings** (the answer + key evidence), so the coordinator synthesizes summaries, not raw logs.
- Isolation also defeats **review bias**: a fresh-context reviewer/evaluator (S5 multi-pass review, evaluator-optimizer) doesn't inherit the author's reasoning, so it catches what same-session self-review (anti-pattern #9) would rationalize away.

**Exam signal (S3/S5):** "coordinator's window fills up / too much detail flows back" → subagents must return *summaries*; the coordinator must *not* absorb every transcript. Trap: one mega-agent doing all subtasks in a single shared window (blows context, serializes parallel work).

## Context Rot — Signs & Remedies

**Context rot** = degraded model behavior as the window fills with long, noisy, or stale content. More tokens is not strictly better; signal-to-noise matters, and quality/instruction-following can fall well before the hard limit.

**Signs to recognize in a stem:**

- Instruction-following slips late in a session (it stops honoring the system prompt / output format it followed earlier).
- The model fixates on stale tool output or an old, now-irrelevant detail.
- It re-asks for or re-derives information already established.
- Repetition, looping, or contradicting earlier correct conclusions.
- Quality drops specifically *as the conversation grows long* (the tell that it's rot, not a prompt bug).

**Remedies (map to CALM):**

- **Compact** verbose history into a tight summary (Lifecycle).
- **Clear stale tool results / context-edit** so dead output exits the window (Lifecycle).
- **Offload to external memory** and retrieve on demand instead of carrying everything (Memory).
- **Isolate** noisy exploration into subagents so the main window stays clean (Isolation).
- **Re-pin the objective** after compaction so the goal stays salient.
- **Curate retrieval** — fewer, higher-relevance documents beat dumping the whole corpus (Allocation).

**Exam signal:** "the agent was fine early and degraded over a long run." That phrasing is the rot tell. The wrong answers are "raise temperature / lower temperature," "add more examples," or "use a bigger model." The right answer manages the *lifecycle* of the window: compact, clear stale results, offload to memory, isolate, re-pin the goal.

## One-Screen CALM Recap

- **C**ontext awareness — token-count what's in the window; know the model's limit.
- **A**llocation — budget top-down: output + thinking headroom, fixed overhead (system+tools), static context, working budget; never plan to fill the window.
- **L**ifecycle — compact on thresholds; preserve objective + plan + key facts; clear stale tool results.
- **M**emory — write structured notes to an external store; retrieve the slice you need; don't rely on volatile history for long-term state.
- **Caching** — order static→dynamic (tools → system → examples/static → messages); ≤4 breakpoints; prefix match; 5-min refreshing TTL (or 1-hour); any change before a breakpoint invalidates everything downstream.
- **Isolation** — subagents in fresh windows return condensed results; fresh-context review beats same-session self-review.
