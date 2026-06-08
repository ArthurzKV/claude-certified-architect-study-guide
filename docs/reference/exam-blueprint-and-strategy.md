# Claude Certified Architect – Foundations (CCA-F): Exam Blueprint and Test-Taking Strategy

This reference is a personal self-study reference for the **Claude Certified Architect – Foundations (CCA-F)** exam — Anthropic's first technical certification, announced March 12 2026 alongside the Claude Partner Network. It is original study material, NOT a dump of live or proctored exam questions. Use it to map the exam structure, budget your time, and learn how to dismantle scenario questions. Pair it with the deeper files in `domains/`, `scenarios/`, `primary-source-synthesis/`, `practice-questions/`, and `reference/`.

## Confirmed Exam Format and Scoring Facts

Memorize these cold — they shape every pacing and risk decision you make on test day.

**Question count and timing.** The exam is **60 multiple-choice, scenario-based questions** delivered in a **120-minute** window. That is a hard average of **2 minutes per question**. Every question is anchored to a production scenario; there are no standalone trivia items.

**Scoring.** Scores are reported on a **scaled 100–1000** range. The **passing score is 720 / 1000**. The scale is non-linear (raw-to-scaled), so do not assume 720/1000 equals exactly 72% correct — treat it as "clearly more right than wrong, with the hard items decided correctly."

**No penalty for guessing.** There is **no deduction for wrong answers**, so you must **answer every question** — never leave a blank. A blind guess has positive expected value; an educated guess (after eliminating one or two distractors) is much better.

**Scenario rotation.** **6 production scenarios** exist in the pool; **4 are randomly selected** for any given sitting. You will not know in advance which four you get, so study **all six**. Every question you see hangs off one of the four scenarios active in your session.

**Audience and intent.** The exam is **partner-oriented** (aligned to the Claude Partner Network) and targets **solution architects with 6+ months of hands-on Claude experience**. Questions reward practical architectural judgment over memorized syntax — they ask "what is the *right* design," not "recite the parameter name."

**Exam signal:** Format facts themselves are not tested, but they govern strategy. The "6+ months hands-on" framing means correct answers reflect lived production patterns, and the most "clever" or maximalist option is usually the trap.

## The Five Domains, Weights, and What Each Tests

The 60 questions are distributed across five domains. Weight tells you where to invest study time and how many questions to expect (a rough count out of 60).

### D1 — Agentic Architecture & Orchestration (27%, ~16 questions)
Tests how you structure agent loops, coordinator/subagent decomposition, stopping conditions, and when to add an agent versus a tool versus a workflow. This is the **largest domain** — master the agent loop and the `stop_reason` contract first. Deep file: `domains/d1-agentic-architecture-orchestration.md`.

### D2 — Tool Design & MCP Integration (18%, ~11 questions)
Tests tool schema design, error-return design, tool-set sizing, and Model Context Protocol (MCP) server/client boundaries. The recurring theme: a tight, well-described tool set with informative errors. Deep file: `domains/d2-tool-design-mcp.md`.

### D3 — Claude Code Configuration & Workflows (20%, ~12 questions)
Tests `CLAUDE.md` memory, plan mode, slash commands, hooks, permissions/settings, subagents, and CI/headless usage. The theme: deterministic enforcement (hooks/config) beats prompt pleading. Deep file: `domains/d3-claude-code-config-workflows.md`.

### D4 — Prompt Engineering & Structured Output (20%, ~12 questions)
Tests prompt structure, tool_use for JSON, structured outputs, extended thinking, and validation-retry loops. The theme: enforce shape with the schema/tool layer, not with "please return valid JSON." Deep file: `domains/d4-prompt-engineering-structured-output.md`.

### D5 — Context Management & Reliability (15%, ~9 questions)
Tests prompt caching, context windows, compaction, batch processing, evals, and per-slice failure analysis. The theme: reliability comes from deterministic checks and sliced metrics, not vibes. Deep file: `domains/d5-context-management-reliability.md`.

**Exam signal:** Many questions live at a domain boundary (e.g., a tool that should be a subagent = D1+D2). Decide which domain *owns the failure mode* in the prompt, then apply that domain's right-pattern.

## The Six Production Scenarios

Every question is anchored to one of these. Study the dedicated file for each in `scenarios/`.

- **S1 — Customer Support Resolution Agent.** Agent SDK, MCP tools, escalation logic. Watch for sentiment-based or confidence-score-based escalation traps. (`scenarios/s1-customer-support-resolution-agent.md`)
- **S2 — Code Generation with Claude Code.** `CLAUDE.md`, plan mode, slash commands. Watch for prompt-based enforcement of rules that belong in hooks. (`scenarios/s2-code-generation-claude-code.md`)
- **S3 — Multi-Agent Research System.** Coordinator/subagent patterns, error handling. Watch for swallowed errors and natural-language stop detection. (`scenarios/s3-multi-agent-research-system.md`)
- **S4 — Developer Productivity with Claude.** Built-in tools, MCP servers, codebase exploration. Watch for over-stuffed tool sets. (`scenarios/s4-developer-productivity.md`)
- **S5 — Claude Code for CI/CD.** Structured output, batch API, multi-pass review. Watch for same-session self-review and aggregate-only metrics. (`scenarios/s5-claude-code-cicd.md`)
- **S6 — Structured Data Extraction.** JSON schemas, tool_use, validation-retry. Watch for prompt-only JSON enforcement and aggregate accuracy masking per-type failures. (`scenarios/s6-structured-data-extraction.md`)

**Exam signal:** The scenario sets the *constraints* (latency budget, error tolerance, who the user is). The right answer must satisfy the scenario's constraints — an option that is technically valid but violates the stated budget is wrong.

## Time-Budget and Flagging Plan

You have 120 minutes for 60 questions — **2 minutes each**, with no slack. Run it in passes.

**Pass 1 (target ~80–90 min): answer everything once.** Spend up to ~90 seconds per question. If you can answer confidently, lock it in. If you stall past ~90 seconds, **pick your best current guess (never blank), flag it, and move on.** Banking the easy questions protects your floor.

**Pass 2 (~20–25 min): revisit flags.** Return to flagged items with fresh eyes. Often a later question's scenario context clarifies an earlier one. Apply elimination deliberately here.

**Pass 3 (~5–10 min): final sweep.** Confirm **zero blanks** (no guessing penalty means a blank is strictly worse than a guess). Resist second-guessing locked answers unless you have a concrete reason — first-instinct changes net negative when the change is just nerves.

**Exam signal:** Scenario preambles are reused across several questions. Read each scenario's setup **once, carefully**, then answer its question cluster without re-reading the whole preamble each time — that is where you recover the seconds you need.

## How to Read a Scenario Question (4-Step Method)

Apply this to every item. It converts vague unease into a decision.

**Step 1 — Identify the domain and the asset under test.** Is this about the agent loop (D1), a tool/MCP boundary (D2), Claude Code config (D3), prompt/output shape (D4), or reliability/context (D5)? Naming the domain summons its right-pattern.

**Step 2 — Identify the trap / anti-pattern.** Scenario distractors are built from the ten classic anti-patterns below. Scan the options for one that "sounds responsible" but secretly relies on natural-language signals, prompt pleading, self-reported confidence, sentiment, swallowed errors, or aggregate-only metrics. That option is usually the *wrong* answer planted to catch the unprepared.

**Step 3 — Apply "the simplest architecture that works."** Anthropic's own guidance ("Building Effective Agents") is to prefer the **least complex** design that meets the requirement: a single prompt before a tool, a tool before a subagent, a workflow before a free-running agent, deterministic code before an LLM call. If two options both work, the **simpler, more deterministic** one is almost always correct.

**Step 4 — Eliminate the anti-pattern distractor, then choose.** Remove the option that embodies a known trap. Of what remains, choose the one that uses a **structured/deterministic control** (e.g., `stop_reason`, a hook, a schema, a per-slice eval) over a soft/natural-language one. When torn between two survivors, prefer the one that fails *loudly and observably*.

**Exam signal:** Questions often phrase the trap as the "easy" or "automatic" choice ("just have the model tell you when it's done"). The structured alternative ("check the API `stop_reason`") is the keeper.

## The Ten High-Yield Anti-Patterns (The Classic Wrong Answers)

These are the distractor factory. If an option matches one of these, it is almost certainly the wrong answer. Full treatment in `reference/anti-patterns-catalog.md`.

1. **Natural-language loop termination** — parsing "I'm done" instead of checking the API **`stop_reason`**. Termination must be a structured signal, never prose.
2. **Iteration caps as the primary stop** — a max-turns cap is a **safety backstop**, not the control mechanism. The real control is the `stop_reason` / task-completion contract.
3. **Prompt-based enforcement of critical business rules** — pleading in the prompt instead of enforcing with **deterministic hooks/code**. Critical rules must be code, not requests.
4. **Self-reported confidence driving escalation** — the model's stated confidence is unreliable; escalate on **deterministic conditions** (tool failure, missing data, policy match).
5. **Sentiment-based escalation** — customer sentiment ≠ task complexity. Escalate on whether the task is *solvable with available tools/permissions*, not on tone.
6. **Generic error messages** that hide diagnostic context from the model — errors should be **specific and actionable** so the model can self-correct.
7. **Silently suppressing errors** — returning empty results as if successful. Failures must **surface**, never masquerade as success.
8. **Too many tools per agent** (e.g., 18) — curate to a **tight ~4–7 tool set**; split overflow via subagents or MCP-server boundaries.
9. **Same-session self-review** — the reviewer inherits the author's bias. Use a **fresh context / separate agent** for review.
10. **Aggregate accuracy metrics that mask per-slice failures** — evaluate **per document type / per category**, not just overall. A 95% aggregate can hide a 40% slice.

**Exam signal:** A single question often plants two anti-pattern distractors and one "almost right" option that fixes the wrong layer. The correct answer fixes the **root cause at the deterministic layer**.

## High-Frequency Facts to Memorize Cold

- **Stop the loop on `stop_reason`.** When the model wants to call a tool, `stop_reason` is **`tool_use`**; continue the loop, run the tool, and return a `tool_result`. The agent loop ends when the model returns a normal completion (e.g., `end_turn`), not when it *says* it is finished.
- **Tool errors go back as `tool_result` with `is_error: true`** and a **descriptive message** — let the model see and recover from the failure rather than swallowing it.
- **Tool-set size:** target **~4–7** tools per agent context; beyond that, accuracy degrades — split via subagents/MCP servers.
- **Prompt caching** trades a higher write cost for a much cheaper read on cache hits; structure prompts with **stable content first** (system, tools, long context) and volatile content last to maximize cache hits.
- **Batch API** is the right tool for **high-volume, latency-tolerant** work (offline evals, bulk extraction, CI fan-out) at lower cost — not for interactive paths.
- **Structured outputs / tool_use for JSON:** enforce shape with the **schema/tool layer plus validation-retry**, never with prompt pleading alone. On a schema-validation failure, **return the validation error and retry**.
- **Extended thinking** helps on hard multi-step reasoning; it is a budgeted capability, not a default for every call.
- **MCP** standardizes how tools/data/context are exposed to models via servers (stdio/HTTP) — it is the integration boundary that lets you split tools cleanly across servers.
- **Plan mode** (Claude Code) proposes a plan for approval before edits; **hooks** enforce deterministic gates; **`CLAUDE.md`** carries durable project memory/conventions.
- **Per-slice evaluation** is mandatory for extraction/classification reliability — report accuracy **by category**, not just the aggregate.

**Exam signal:** Where the prompt invites you to invent an exact number or header, the safe answer describes the **behavior** (e.g., "continue while `stop_reason` is `tool_use`") rather than a fabricated constant.

## 12-Week Study Plan (Mapped to This Corpus)

A domain-weighted ramp. Each week names the files to work and an output to produce. Files live in `primary-source-synthesis/`, `domains/`, `scenarios/`, `practice-questions/`, and `reference/`.

**Week 1 — Orientation & format.** Read this file plus `NOTICE.md` and `reference/decision-frameworks.md`. Internalize format facts and the 4-step reading method. Output: the ten anti-patterns from memory.

**Week 2 — Foundations of agents.** `primary-source-synthesis/building-effective-agents.md` + `primary-source-synthesis/claude-agent-sdk.md`. Output: a one-page agent-loop diagram keyed on `stop_reason`.

**Week 3 — D1 deep (part 1).** `domains/d1-agentic-architecture-orchestration.md` (loops, stopping conditions) + `primary-source-synthesis/tool-use-fundamentals.md`. Output: distinguish stop_reason control vs. iteration-cap backstop in your own words.

**Week 4 — D1 deep (part 2) + S3.** Coordinator/subagent patterns; `scenarios/s3-multi-agent-research-system.md`. Output: when to add a subagent vs. a tool vs. a workflow.

**Week 5 — D2 tools & MCP.** `domains/d2-tool-design-mcp.md` + `primary-source-synthesis/mcp-deep-reference.md`. Output: a tool schema with informative `is_error` returns; justify a 4–7 tool budget.

**Week 6 — D2 + S1.** `scenarios/s1-customer-support-resolution-agent.md`. Output: a deterministic escalation rule (no sentiment, no self-confidence).

**Week 7 — D3 Claude Code.** `domains/d3-claude-code-config-workflows.md` + `primary-source-synthesis/claude-code-deep-reference.md`. Output: a `CLAUDE.md` + one hook that enforces a rule deterministically.

**Week 8 — D3 + S2 & S4.** `scenarios/s2-code-generation-claude-code.md` and `scenarios/s4-developer-productivity.md`. Output: plan-mode workflow + a curated MCP/tool set for codebase exploration.

**Week 9 — D4 prompts & structured output.** `domains/d4-prompt-engineering-structured-output.md`, `primary-source-synthesis/prompt-engineering-techniques.md`, `primary-source-synthesis/structured-outputs.md`, `primary-source-synthesis/extended-thinking.md`. Output: a tool_use JSON spec with a validation-retry loop.

**Week 10 — D4 + S6.** `scenarios/s6-structured-data-extraction.md`. Output: per-type eval plan that would catch a masked slice failure.

**Week 11 — D5 context & reliability.** `domains/d5-context-management-reliability.md`, `primary-source-synthesis/prompt-caching.md`, `primary-source-synthesis/batch-scale-and-models.md`, `primary-source-synthesis/context-engineering-and-calm.md`, `primary-source-synthesis/evals-and-reliability.md`, `primary-source-synthesis/contextual-retrieval-rag.md`. Plus `scenarios/s5-claude-code-cicd.md` (multi-pass review, fresh-context reviewer). Output: a caching layout + per-slice eval dashboard sketch.

**Week 12 — Drill & simulate.** Work `practice-questions/rapid-fire-flashcards.md` to recall, then run timed mixed sets across all six scenarios at **2 min/question**. Re-read `reference/anti-patterns-catalog.md`. Output: a full timed mock; review every miss back to its anti-pattern and domain.

**Exam signal:** In the final two weeks, practice the *reading method* under time pressure, not just content recall — on exam day the bottleneck is decision speed, not knowledge gaps.

## Sources

- Anthropic, "Building Effective Agents" — https://www.anthropic.com/research/building-effective-agents
- Claude Agent SDK & API docs — https://platform.claude.com/docs and https://docs.anthropic.com/en/api
- Claude Code documentation — https://docs.anthropic.com/en/docs/claude-code
- Model Context Protocol — https://modelcontextprotocol.io
- Anthropic engineering on context, evals, and prompt caching — https://www.anthropic.com/engineering
