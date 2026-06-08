# Decision Frameworks Cheat-Sheet (CCA-F)

Pure reference. This file is a memorizable set of **IF … THEN …** decision rules and small decision trees that map a scenario stem to the correct architecture choice. Every rule here reflects publicly-documented Anthropic behavior from "Building Effective Agents" (https://www.anthropic.com/research/building-effective-agents), the multi-agent research write-up (https://www.anthropic.com/engineering/built-multi-agent-research-system), the Claude API docs (https://platform.claude.com/docs and https://docs.claude.com/en/docs), Claude Code docs (https://docs.anthropic.com/en/docs/claude-code), and the MCP spec (https://modelcontextprotocol.io). When a scenario question gives you a stem, find the matching discriminator below and pick the row.

The meta-rule that overrides everything: **use the simplest architecture that works, and only add autonomy when the task genuinely requires it.** Added autonomy buys capability at the cost of latency, tokens, and reliability — pay it only when it buys something.

## Decision: Single Call vs Workflow vs Agent

Walk this tree top-to-bottom and stop at the first match.

- **IF** one prompt (optionally with retrieval / in-context examples) can produce the answer → **single LLM call**. Don't orchestrate.
- **ELSE IF** the steps are *known and enumerable in advance* (fixed sequence, fixed categories, fixed parallel slices) → **workflow** (predefined code paths; model fills the steps).
- **ELSE IF** the model must decide *which* steps and *how many* at runtime, and you cannot enumerate them up front → **agent** (model dynamically directs its own loop).

**Exam signal:** "First classify, then look up the account, then draft a reply" = enumerable = **workflow**, even if a distractor dresses it up as a multi-agent system. "Variable-depth research" / "touches an unknown set of files" / "the model decides when it's done" = **agent**. The flashy multi-agent option for an enumerable task is a trap.

## Decision Tree: Which of the 5 Workflow Patterns

One discriminator picks the pattern. Memorize the right column.

| IF the stem says… | THEN pattern | Why |
|---|---|---|
| Fixed sequence; each step consumes the prior output | **Prompt chaining** | Decompose into simpler steps; add gate checks between them |
| Distinct input categories each better handled separately | **Routing** | Classify-then-dispatch; canonical cost-optimization (route easy→Haiku, hard→Opus/Sonnet) |
| Independent subtasks that can run at the same time | **Parallelization (sectioning)** | Lower wall-clock latency; e.g. review code for security/perf/style separately |
| Same task run multiple times for consensus / safety | **Parallelization (voting)** | Redundancy raises reliability; e.g. N independent harm checks |
| Subtasks unknown until runtime; orchestrator invents them | **Orchestrator-workers** | Dynamic decomposition; underpins S3 |
| Clear rubric + iterative refinement measurably helps | **Evaluator-optimizer** | Generate→critique→refine loop; evaluator must be a **fresh-context** call |

**Trap:** chaining independent steps (wastes latency you could parallelize) and using parallelization for steps that actually depend on each other.

## Decision: Single-Agent vs Multi-Agent (Hub-and-Spoke)

- **IF** the work fits one context window, is mostly sequential, or has tight step-to-step dependencies → **single agent**. Multi-agent here just adds coordination overhead and token cost.
- **IF** the task is breadth-first / parallelizable, needs more effective context than one window holds, or has independent sub-investigations → **hub-and-spoke multi-agent**: a coordinator that **decomposes → delegates → synthesizes**, spawning parallel subagents each in an **isolated context**.

**Hard rule:** the coordinator delegates; it does **not** personally run every tool call. A coordinator that does all the work itself is always wrong (blows context, serializes parallelizable work). Subagents return a **condensed result**, not their full transcript.

**Exam signal (S3):** "scale to many sources / sub-questions" → coordinator + parallel isolated subagents + synthesis. Distractors: one mega-agent with 18 tools doing it all; or a coordinator that executes everything.

## Decision: When to Escalate to a Human

Escalate on **deterministic, task-property signals** — never on vibes.

- **ESCALATE IF** a required tool/permission is unavailable, an action is irreversible/high-stakes (refund over threshold, account deletion), policy requires human sign-off, repeated tool failures, or the request is out of the agent's defined scope.
- **DO NOT escalate based on:** the model's **self-reported confidence score** (anti-pattern #4), or **customer sentiment** (anti-pattern #5 — sentiment ≠ task complexity).

**Exam signal (S1):** the right escalation trigger is a concrete business condition checked in code/hooks (amount > limit, missing entitlement, N retries exhausted). "Escalate when the model says it's unsure" or "when the customer is angry" are the classic wrong answers.

## Decision: Tool vs MCP Server vs Subagent vs Slash Command

| IF you need… | THEN use | Notes |
|---|---|---|
| A single capability/action callable by the model | **Tool** (tool_use) | Name + rich description + tight `input_schema` |
| A reusable, shareable bundle of tools/resources across an external system or many agents | **MCP server** | Standard transport/auth boundary; one server per system (CRM, DB, GitHub) |
| To offload a focused chunk of work into its own isolated context | **Subagent** | Returns condensed result; protects parent context; enables parallelism |
| A user-triggered reusable prompt/macro in Claude Code | **Slash command** | A canned prompt the human invokes, not something the model auto-selects |

**Boundary rule:** group tools into MCP servers by *external system*, and split capability across **subagents** when one agent would otherwise carry too many tools. Slash commands are for the *human's* workflow; tools/MCP are for the *model's* runtime choices.

## Decision: How Many Tools Per Agent

- **TARGET ~4–7 tightly-curated tools** per agent.
- **IF** you're pushing past that (e.g. 18 tools) → **split** the work across subagents or distinct MCP boundaries; don't pile them onto one agent (anti-pattern #8).
- **IF** two tools have overlapping names/purposes (`search`/`find`/`query` on the same data) → **merge or rename** for distinct, non-overlapping semantics to cut selection error.

**Exam signal:** "the agent has 18 tools and keeps picking the wrong one" → curate down and/or split via subagents, *not* "lower temperature" or "add more system-prompt instructions."

## Decision: RAG vs Long-Context vs Agentic Search

| IF… | THEN |
|---|---|
| Large, frequently-changing or huge corpus; need cited, targeted retrieval | **RAG** (retrieve top-k, ground the answer) |
| The whole relevant material fits comfortably in context and is reused | **Long-context** (load it directly; pair with **prompt caching**) |
| The model must *navigate/explore* to find what it needs (grep, read files, follow links, unknown location) | **Agentic search** (give it search/read tools and let it iterate) |

**Exam signal (S4):** "explore an unfamiliar codebase" → **agentic search** with built-in file/grep/read tools, not stuffing the entire repo into one prompt. "Answer over a fixed, modest doc set reused across calls" → long-context + caching.

## Decision: Prompt Caching — Yes/No and Where to Put Breakpoints

- **CACHE IF** a large, **stable prefix** (system prompt, tool definitions, long documents, few-shot examples) is reused across many requests. Skip caching for short, always-unique prompts.
- **Order content static → dynamic.** Put the unchanging bulk first; put per-request variable content last.
- **Place the breakpoint at the boundary between stable and variable content**, after the last reusable block. Everything before the breakpoint is cached; a cache hit requires an **exact prefix match**, so any change before the breakpoint invalidates it.
- Typical breakpoint placement: after tool definitions and the system prompt, and/or after a large shared document — **before** the user's per-turn message.

**Exam signal:** "high token cost from re-sending the same tools/system prompt every call" → enable prompt caching with a breakpoint after the stable prefix. Trap: putting variable content *before* the cached block (kills the hit rate) or caching tiny unique prompts.

## Decision: Sync vs Batch vs Streaming

| IF you need… | THEN | Trade-off |
|---|---|---|
| An interactive, low-latency single response now | **Sync** | Standard request/response |
| Token-by-token output for UX / long generations | **Streaming** | Same latency to first token, better perceived responsiveness; lets you handle output incrementally |
| Large volume, **latency-tolerant**, cost-sensitive offline work | **Batch API** | Much cheaper, async turnaround; not for interactive paths |

**Exam signal (S5):** "score thousands of CI artifacts overnight / bulk offline evaluation" → **Batch API**. "Live chat assistant" → sync/streaming. Choosing batch for an interactive flow (or sync for a huge offline job) is the trap.

## Decision: Which Model (Opus vs Sonnet vs Haiku)

- **Haiku** — highest throughput / lowest cost / fastest. **IF** the task is simple, high-volume, latency-critical, or a cheap router/classifier → Haiku.
- **Sonnet** — balanced default for most production agentic and coding work; strong capability at moderate cost.
- **Opus** — top reasoning depth. **IF** the task is the hardest reasoning, complex multi-step agentic planning, or the orchestrator of a demanding multi-agent system → Opus.

**Routing pattern:** classify difficulty first, then dispatch easy→Haiku, hard→Opus/Sonnet (this is the routing workflow doing cost control). Don't run Opus on trivially-classifiable volume; don't put Haiku on the hardest reasoning step.

## Decision: Structured-Output Method Selection

| IF… | THEN method |
|---|---|
| You need a guaranteed object shape and want the model to "call a function" with typed args | **tool_use with a strict JSON Schema** (single forced tool) |
| The platform/SDK exposes a native structured-outputs / response-format mode | **Structured outputs** (schema-constrained decoding) |
| You only need light shaping and will validate yourself | **Prompt-instructed JSON** (least reliable — validate + retry) |

**Validation-retry loop (S6):** parse → validate against the schema → on failure, **feed the specific validation error back** to the model and retry. Surfacing the concrete error (not a generic message) is what makes the retry converge (counters anti-patterns #6/#7). Aggregate-only accuracy that hides per-document-type failures is anti-pattern #10 — evaluate **per slice**.

## Decision: When to Use Extended Thinking

- **USE extended thinking IF** the task needs genuine multi-step reasoning, planning, math/logic, or complex tool-use orchestration where visible deliberation improves correctness.
- **SKIP IT IF** the task is simple lookup/extraction/formatting, latency-critical, or high-volume cheap work — thinking adds tokens and latency for no gain.
- Pair extended thinking with the harder model tiers and with agentic planning steps; don't bolt it onto a Haiku classifier.

**Exam signal:** "complex debugging / multi-constraint planning where the agent skips steps" → enable extended thinking. "Fast, simple, high-volume" → leave it off.

## Decision: Hooks vs Prompt vs Permissions for Enforcement

This is the highest-frequency D3 discriminator. **Critical/deterministic rules must be enforced in code, never pleaded in the prompt.**

| IF the rule is… | THEN enforce with |
|---|---|
| A hard business/safety invariant that must *always* hold (format, gating, blocking a dangerous command, post-edit checks) | **Hook** (deterministic, runs every time) |
| Soft guidance, style, preference, "try to…" | **Prompt** (system prompt / CLAUDE.md) |
| Controlling *what the agent is allowed to touch/run* (file scopes, command allow/deny, network) | **Permissions** |

**Rules:**
- **IF** "the model sometimes ignores the instruction and it matters" → move it from prompt to a **hook** (anti-pattern #3: prompt-pleading for critical rules is wrong).
- **IF** "prevent the agent from running/deleting/accessing X" → **permissions**, not a prompt request.
- Prompts persuade; hooks guarantee; permissions constrain. Match the mechanism to how much it matters.

## Decision: Loop Termination & Reliability Guards

- **TERMINATE the agent loop on the API `stop_reason`** (e.g. the model stops requesting tools / returns `end_turn`), **not** by parsing natural language like "I'm done" (anti-pattern #1).
- **Iteration caps are a safety backstop, not the primary control** (anti-pattern #2). The real stop is the deterministic stop signal; the cap only prevents runaway.
- **Never silently suppress errors** or return empty results as success (anti-pattern #7). Surface **diagnostic** error messages back to the model (counters anti-pattern #6).
- **Reviewers/evaluators run in a fresh context / separate agent** so they don't inherit the author's reasoning bias (anti-pattern #9).

**Exam signal:** any answer that relies on a magic iteration number *as the control*, parses prose to decide "done," hides errors, or self-reviews in the same session is a distractor. Pick the deterministic-signal option.

## One-Line Discriminator Index (rapid recall)

- Enumerable steps → **workflow**; model picks steps at runtime → **agent**; one prompt suffices → **single call**.
- Independent slices → **parallelization**; fixed sequence → **chaining**; categorize-then-specialize → **routing**; runtime-unknown subtasks → **orchestrator-workers**; rubric + iterate → **evaluator-optimizer**.
- Coordinator **delegates**, never does it all; subagents are **isolated** and return **condensed** results.
- Escalate on **task-property/business** signals; never on **confidence** or **sentiment**.
- ~**4–7 tools** per agent; over that → **split via subagents/MCP**.
- Explore-to-find → **agentic search**; reused fixed set → **long-context + caching**; huge changing corpus → **RAG**.
- Cache the **stable prefix**, breakpoint at the static→dynamic boundary, static content **first**.
- Offline bulk + latency-tolerant → **Batch**; interactive → **sync/streaming**.
- Hardest reasoning → **Opus**; balanced default → **Sonnet**; cheap/fast/high-volume → **Haiku**.
- Guaranteed shape → **tool_use / structured outputs** + **validation-retry feeding the real error**.
- Must-always-hold rule → **hook**; soft guidance → **prompt**; what it can touch → **permissions**.
- Stop on **`stop_reason`**, not prose; iteration cap is a **backstop**; never hide errors; review in **fresh context**.
