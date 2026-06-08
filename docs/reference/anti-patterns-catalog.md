# CCA-F Anti-Patterns Catalog: The Definitive Guide to Wrong Answers

This catalog enumerates the high-yield **anti-patterns** that appear as distractors on the Claude Certified Architect – Foundations (CCA-F) exam. Every scenario question pairs one correct architectural choice against several "tempting-but-wrong" options. Learning to recognize the anti-pattern is often faster than deriving the right answer from scratch. For each entry: the **tempting wrong approach**, **why it fails**, the **correct pattern**, and the **domain/scenario** where it surfaces.

This is original study material grounded in publicly documented Anthropic behavior (platform.claude.com/docs, docs.anthropic.com/en/docs/claude-code, modelcontextprotocol.io, and Anthropic's engineering/research posts). It is NOT an exam dump.

---

## How to Use This Catalog on the Exam

When a question gives four options, scan for the anti-pattern signatures below. The wrong answers cluster around four failure themes: (a) **non-determinism where determinism is required**, (b) **the model judging itself**, (c) **hiding information from the model or the operator**, and (d) **eager work instead of just-in-time work**. If an option relies on the model's good intentions to enforce a hard rule, it is almost always wrong.

**Exam signal:** Distractors are written to sound reasonable and "agentic." The correct answer is usually the more boring, more deterministic, more observable one.

---

## Anti-Pattern 1: Natural-Language Loop Termination Instead of stop_reason

**Tempting wrong approach:** Detect when the agent is finished by string-matching its prose — looking for "I'm done," "task complete," or "final answer" in the text output, then breaking the loop.

**Why it fails:** Model phrasing is non-deterministic and locale/prompt-dependent. The model may say "done" mid-thought, or finish without ever saying it. The Messages API already returns a structured `stop_reason` (e.g., `end_turn`, `tool_use`, `max_tokens`, `pause_turn`). The agent loop should continue while `stop_reason == "tool_use"` and terminate on `end_turn`. Parsing prose is brittle and silently breaks.

**Correct pattern:** Drive the loop on the API's `stop_reason` field. Continue when the model requests a tool (`tool_use`), execute the tool, append the `tool_result`, and re-call. Stop on `end_turn`.

**Where it appears:** D1 Agentic Architecture (loop control), S3 Multi-Agent Research, S1 Support Agent.

---

## Anti-Pattern 2: Iteration Cap as the Primary Stopping Mechanism

**Tempting wrong approach:** Control the agent by setting `max_iterations = 10` and treating that ceiling as the way the agent knows when to stop.

**Why it fails:** An iteration cap is a **safety backstop** against runaway loops and cost — not a completion signal. Tasks that finish in 3 turns waste nothing, but tasks that legitimately need 12 turns get truncated mid-work, producing partial/incorrect results. The cap masks whether the agent actually completed the goal.

**Correct pattern:** Terminate on genuine completion signals (`stop_reason == "end_turn"`, a verified success condition, or a tool that reports "goal met"). Keep the iteration cap *as well*, but as a circuit breaker that logs an anomaly when hit — not as the control flow.

**Where it appears:** D1 Agentic Architecture, D5 Context Management & Reliability, S3 Multi-Agent Research.

---

## Anti-Pattern 3: Prompt-Based Enforcement of Critical Business Rules

**Tempting wrong approach:** Enforce a hard requirement ("never issue a refund over $500 without manager approval," "always redact PII") by writing it forcefully into the system prompt — "You MUST NEVER...".

**Why it fails:** Prompts shape probability, not guarantees. A sufficiently adversarial input, an edge case, or simple sampling variance can produce a violation. Critical business rules, security boundaries, and compliance controls must be **deterministic**. Claude Code's hooks and tool-level validation exist precisely so enforcement happens in code, not in pleading text.

**Correct pattern:** Enforce in deterministic code — a Claude Code hook (PreToolUse/PostToolUse), a tool wrapper that validates arguments before executing, or a server-side guard. Let the prompt *describe* the rule for good UX, but never let it be the *only* line of defense.

**Where it appears:** D2 Tool Design, D3 Claude Code Workflows (hooks), S1 Support Agent (escalation/refund limits), S5 CI/CD (gating merges).

---

## Anti-Pattern 4: Self-Reported Confidence Scores Driving Escalation

**Tempting wrong approach:** Ask the model to emit a `confidence: 0.0–1.0` field and escalate to a human (or a stronger model) whenever confidence is below a threshold.

**Why it fails:** LLM self-reported confidence is poorly calibrated — models are frequently confidently wrong and occasionally underconfident when correct. Tying a control decision to an uncalibrated self-assessment makes escalation noisy and gameable. It is a self-evaluation in disguise (see #9).

**Correct pattern:** Escalate on **objective, external signals**: a tool/validation failure, a schema-validation retry exhausting its budget, an out-of-policy action requested, missing required data, or a deterministic complexity heuristic (e.g., number of accounts touched). If you need a model judgment, use a *separate* grader with explicit rubric criteria rather than the actor's self-score.

**Where it appears:** D1 Orchestration, D5 Reliability, S1 Support Agent (escalation logic).

---

## Anti-Pattern 5: Sentiment-Based Escalation (Sentiment ≠ Complexity)

**Tempting wrong approach:** Route a support ticket to a human whenever the customer sounds angry or frustrated.

**Why it fails:** Customer sentiment measures *emotional tone*, not *task difficulty*. An angry customer may have a trivially resolvable problem; a polite one may have a complex multi-system issue requiring escalation. Conflating the two sends easy tickets to humans and lets hard tickets slip through automation.

**Correct pattern:** Escalate on **task complexity and authority** signals: required action exceeds the agent's permission scope, multiple systems/accounts involved, policy exception needed, or a tool reports it cannot complete the request. Sentiment can be a *secondary* input for prioritization/tone, never the primary escalation gate.

**Where it appears:** D1 Orchestration, S1 Customer Support Resolution Agent.

---

## Anti-Pattern 6: Generic Error Messages That Hide Diagnostic Context

**Tempting wrong approach:** When a tool fails, return `{"error": "Request failed"}` (or just "An error occurred") to the model.

**Why it fails:** The model is an active agent that can *recover* from errors if it can see them. A generic message strips the information it needs to retry intelligently — which field was invalid, which record was missing, what the rate limit window is. The model then guesses blindly or gives up.

**Correct pattern:** Return **structured, actionable** error context in the `tool_result` (with `is_error: true`): the specific failure, the offending input, and ideally a hint ("date must be ISO-8601," "account 4471 not found; verify the ID"). Give the model what a human debugger would want.

**Where it appears:** D2 Tool Design, D5 Reliability, S6 Structured Data Extraction (validation-retry), S1 Support Agent.

---

## Anti-Pattern 7: Silently Suppressing Errors (Empty Results as Success)

**Tempting wrong approach:** Catch an exception in a tool and return an empty list / empty object as if the call succeeded, so the agent "keeps moving."

**Why it fails:** This is worse than a loud failure. The model interprets the empty result as a *real* answer ("no matching records exist") and proceeds on false data — silently producing wrong conclusions. It violates the root-cause principle: hiding a failure is never a fix.

**Correct pattern:** Surface failures explicitly via `is_error: true` with diagnostic detail. Distinguish "genuinely zero results" from "the query failed." The agent must be able to tell a true empty set from a broken tool.

**Where it appears:** D2 Tool Design, D5 Reliability, S4 Developer Productivity, S6 Data Extraction.

---

## Anti-Pattern 8: Too Many Tools on One Agent

**Tempting wrong approach:** Hand a single agent 18 tools because "it might need any of them," expecting the model to pick correctly.

**Why it fails:** Large tool surfaces degrade selection accuracy, bloat the context window with schemas, and increase the odds of wrong-tool/wrong-argument errors. Anthropic's guidance favors a tight, curated set (~4–7 well-scoped tools) per agent.

**Correct pattern:** Curate tools to the agent's actual job. Split capability across **subagents** (each with its own focused toolset) or behind **MCP server boundaries**, so the coordinator delegates rather than juggling everything. Fewer, sharper tools with clear descriptions beat a giant menu.

**Where it appears:** D1 Orchestration, D2 Tool Design & MCP, S3 Multi-Agent Research, S4 Developer Productivity.

---

## Anti-Pattern 9: Same-Session Self-Review

**Tempting wrong approach:** After the agent writes code (or an answer), ask the *same* agent in the *same* context to review its own work for bugs.

**Why it fails:** The reviewer inherits the author's reasoning, assumptions, and blind spots. It tends to rationalize and confirm rather than find flaws — the bias that produced the bug is still in context. This is the structural reason self-confidence (#4) is untrustworthy too.

**Correct pattern:** Review in a **fresh context** — a separate agent/subagent (or a new session) with no access to the author's chain of thought, given only the artifact and an explicit review rubric. In CI/CD, this is a distinct review pass. Independence is the whole point.

**Where it appears:** D1 Orchestration, D3 Claude Code Workflows, S5 CI/CD (multi-pass review), S3 Multi-Agent Research.

---

## Anti-Pattern 10: Aggregate Accuracy That Masks Per-Slice Failure

**Tempting wrong approach:** Validate an extraction/agent system by reporting one number — "94% accuracy overall" — and shipping on it.

**Why it fails:** A single aggregate hides systematic failure in subpopulations. 94% overall can mean 99% on the common document type and 40% on invoices, or 100% on English and 50% on a second language. The category that fails is invisible until production.

**Correct pattern:** Evaluate **per slice** — per document type, per category, per language, per field. Track the worst-performing slice, not just the mean. An eval set must be stratified so each meaningful subpopulation has coverage.

**Where it appears:** D4 Prompt Engineering & Structured Output, D5 Reliability, S6 Structured Data Extraction, S5 CI/CD.

---

## Additional High-Yield Anti-Patterns

### Anti-Pattern 11: Secrets in CLAUDE.md (or Other Committed Context)

**Tempting wrong approach:** Paste API keys, tokens, or DB credentials into `CLAUDE.md` so the agent "has what it needs."

**Why it fails:** `CLAUDE.md` is committed to the repo, loaded into context on every session, and easily exfiltrated. Secrets belong in environment variables / secret managers, never in prompt-loaded files. This is also a context-bloat and security-boundary violation.

**Correct pattern:** Reference secrets via environment variables or a secrets manager; keep `CLAUDE.md` to durable project conventions, commands, and architecture. **Exam signal:** any option that puts credentials in a context file is wrong.

**Where it appears:** D3 Claude Code Configuration, S2 Code Generation.

---

### Anti-Pattern 12: Over-Broad MCP Credentials / Scope

**Tempting wrong approach:** Give an MCP server admin/root credentials or a wildcard scope so it "won't be blocked" by missing permissions.

**Why it fails:** Violates least privilege. An over-scoped MCP server is a large blast radius — a prompt injection or buggy tool can now delete, exfiltrate, or modify far beyond its job. MCP integrations should be scoped to exactly the resources the task needs.

**Correct pattern:** Grant each MCP server the **minimum** scopes/permissions required; use read-only credentials where writes aren't needed; separate servers by trust boundary. **Exam signal:** "grant full access to avoid permission errors" is a trap.

**Where it appears:** D2 Tool Design & MCP Integration, S1 Support Agent, S4 Developer Productivity.

---

### Anti-Pattern 13: Parsing Free-Text Output Instead of Structured Output / tool_use

**Tempting wrong approach:** Ask Claude to "return the data as JSON" in prose, then regex/`json.loads` the text, hoping it's well-formed.

**Why it fails:** Free-text JSON is unreliable — prose preambles, code fences, trailing commentary, or a stray field break the parse. For machine-consumed data you want a guaranteed shape.

**Correct pattern:** Use **tool_use with a JSON Schema** (the tool's `input_schema`) so the model emits structured arguments, or use the structured-output mechanism. Validate against the schema and retry on failure. **Exam signal:** when output feeds another program, structured > prose parsing.

**Where it appears:** D4 Structured Output, D2 Tool Design, S6 Data Extraction, S5 CI/CD.

---

### Anti-Pattern 14: Pre-Loading the Entire Knowledge Base Instead of Just-in-Time Retrieval

**Tempting wrong approach:** Stuff the whole document corpus / codebase / knowledge base into context up front so "the model has everything."

**Why it fails:** Wastes tokens, dilutes attention (relevant facts buried among irrelevant ones), raises latency and cost, and can exceed the window. More context is not more capability — signal-to-noise matters.

**Correct pattern:** Retrieve **just-in-time** — give the agent search/retrieval tools (or MCP resources) and let it pull only what each step needs. Use the file system / external store as memory; load on demand. **Exam signal:** "load all the docs into the prompt" loses to "give it a retrieval tool."

**Where it appears:** D5 Context Management, D1 Orchestration, S4 Developer Productivity, S3 Research.

---

### Anti-Pattern 15: Not Preserving Thinking Blocks Across Tool Turns

**Tempting wrong approach:** With extended thinking enabled, strip or drop the `thinking` blocks from the assistant turn before sending the next request after a tool call.

**Why it fails:** When extended thinking is used with tool use, the API expects prior `thinking` (and `redacted_thinking`) blocks to be **passed back unmodified** in the conversation so the model can continue its reasoning coherently. Dropping or editing them breaks signature verification / reasoning continuity and degrades multi-turn tool performance.

**Correct pattern:** Preserve and return thinking blocks verbatim across tool-use turns exactly as received. Don't reorder or rewrite them. **Exam signal:** any option that "removes the thinking to save tokens" during a tool loop is wrong.

**Where it appears:** D4 Prompt Engineering (extended thinking), D5 Context Management, S3 Research, S5 CI/CD.

---

### Anti-Pattern 16: Cache Breakpoint Placed Before Volatile Content

**Tempting wrong approach:** Put the prompt-caching breakpoint near the end of the prompt — after the user's per-request question or other content that changes every call.

**Why it fails:** Prompt caching keys on the **exact prefix** up to the breakpoint. If volatile content sits inside the cached prefix, the prefix changes every request and you get near-zero cache hits — paying to write the cache without ever reading it.

**Correct pattern:** Order the prompt **stable → volatile**: put the large, unchanging material (system prompt, tool definitions, big documents) first and place the cache breakpoint *after* the stable block but *before* the volatile per-request content. Cache the prefix that actually repeats. **Exam signal:** look for the option that caches static-then-dynamic in the right order.

**Where it appears:** D5 Context Management (prompt caching), D4 Prompt Engineering, S1 Support Agent, S5 CI/CD.

---

### Anti-Pattern 17: Misusing Temperature / Forced Tool Choice With Extended Thinking

**Tempting wrong approach:** Turn on extended thinking and also set a non-default `temperature`, or force `tool_choice` to a specific/required tool, expecting all three to compose freely.

**Why it fails:** Extended thinking has constraints — it requires the default temperature setting and is **incompatible with forced tool use** (`tool_choice` of type `tool`/`any`). Forcing a tool removes the model's ability to reason first about whether/which tool to call, which is the entire value of thinking. Configuring them together produces errors or defeats the feature.

**Correct pattern:** With extended thinking, leave temperature at its required default and use `tool_choice: auto` so the model reasons before deciding on tools. Don't combine thinking with `top_p`/`top_k` tweaks or forced tool selection. **Exam signal:** the distractor stacks "extended thinking + temperature 0.2 + forced tool."

**Where it appears:** D4 Prompt Engineering & Structured Output, D2 Tool Design, S6 Data Extraction, S5 CI/CD.

---

### Anti-Pattern 18: Skipping the Validation-Retry Loop on Structured Output

**Tempting wrong approach:** Trust the first structured/JSON response and pass it downstream without schema validation.

**Why it fails:** Even with tool_use schemas, a model can omit a required field, mistype an enum, or violate a constraint the schema can't fully express. One bad record corrupts the pipeline.

**Correct pattern:** Validate every structured output against the schema; on failure, **feed the validation error back** to the model and retry (bounded retries). Escalate after the retry budget is exhausted — not silently. This pairs with #6 (give it the actual error) and #4 (escalate on objective failure, not self-confidence).

**Where it appears:** D4 Structured Output, D5 Reliability, S6 Structured Data Extraction.

---

### Anti-Pattern 19: Flat Multi-Agent Topology Instead of Coordinator/Subagent

**Tempting wrong approach:** Spawn many peer agents that all read/write the same shared context and message each other freely, with no orchestrator.

**Why it fails:** Shared mutable context creates race conditions, context pollution, and conflicting actions; no single component owns task decomposition or result synthesis. It doesn't scale and is hard to debug.

**Correct pattern:** Use a **coordinator (lead) + subagent** topology: the coordinator decomposes the task, dispatches focused subagents with isolated context windows, and synthesizes their returned results. Each subagent has a narrow toolset (#8) and its own context. **Exam signal:** Anthropic's multi-agent research pattern — orchestrator delegates, subagents work in parallel with clean contexts.

**Where it appears:** D1 Agentic Architecture & Orchestration, S3 Multi-Agent Research System.

---

### Anti-Pattern 20: Batch/Synchronous Mismatch — Using the Wrong API for the Workload

**Tempting wrong approach:** Run thousands of independent, latency-insensitive evaluations as individual synchronous Messages calls (or, conversely, route a real-time user-facing request through the Batch API).

**Why it fails:** The Batch API exists for **large volumes of asynchronous, non-urgent** work at lower cost — perfect for offline evals, bulk extraction, CI multi-pass review over many files. Forcing that through synchronous calls is slow and expensive; forcing an interactive request through batch adds unacceptable latency.

**Correct pattern:** Match the API to the workload: **Batch** for high-volume, throughput-oriented, latency-tolerant jobs; **synchronous Messages** for interactive, low-latency needs. **Exam signal:** "score 10,000 CI test cases overnight" → Batch; "answer a live support chat" → synchronous.

**Where it appears:** D5 Reliability, D3 Claude Code Workflows, S5 CI/CD, S6 Data Extraction.

---

### Anti-Pattern 21: Unbounded Context Growth Instead of Compaction / Summarization

**Tempting wrong approach:** Let a long-running agent accumulate every tool result and message in context indefinitely until it nears or hits the window limit.

**Why it fails:** Performance and relevance degrade as the window fills with stale tool output; cost rises; eventually requests fail. Important early instructions get crowded out.

**Correct pattern:** Manage the context budget — **summarize/compact** older turns, prune large tool outputs once consumed, and offload durable state to external memory (files/store) that can be re-retrieved (#14). Keep only what's needed for the current step. **Exam signal:** "keep appending forever" loses to "compact and externalize memory."

**Where it appears:** D5 Context Management & Reliability, D1 Orchestration, S3 Research, S4 Developer Productivity.

---

## Quick-Reference Trap Table

| # | Anti-pattern | One-line tell | Right move |
|---|---|---|---|
| 1 | NL loop termination | matches "I'm done" | use `stop_reason` |
| 2 | Iteration cap as control | `max_iters` decides done | completion signal; cap = backstop |
| 3 | Prompt-enforced rules | "you MUST never…" only | deterministic hook/code |
| 4 | Self-confidence escalation | `confidence < 0.7` → human | objective failure signals |
| 5 | Sentiment escalation | angry → human | complexity/authority signals |
| 6 | Generic errors | `"error": "failed"` | structured diagnostic detail |
| 7 | Silent error suppression | empty result on failure | `is_error: true`, surface it |
| 8 | Too many tools | 18 tools on one agent | 4–7; split via subagents/MCP |
| 9 | Same-session self-review | author reviews self | fresh-context reviewer |
| 10 | Aggregate-only metric | "94% overall" | per-slice evaluation |
| 11 | Secrets in CLAUDE.md | keys in context file | env vars / secret manager |
| 12 | Over-broad MCP scope | admin creds "to be safe" | least privilege |
| 13 | Parse free-text JSON | regex the prose | tool_use / structured output |
| 14 | Pre-load whole KB | dump all docs in prompt | just-in-time retrieval |
| 15 | Drop thinking blocks | strip thinking in tool loop | preserve verbatim |
| 16 | Cache before volatile | breakpoint after the question | stable→volatile ordering |
| 17 | Thinking + temp/forced tool | thinking + temp 0.2 + forced | default temp, `tool_choice: auto` |
| 18 | No validation-retry | trust first JSON | validate, feed error back, retry |
| 19 | Flat agent topology | peers share mutable context | coordinator + isolated subagents |
| 20 | Wrong API for workload | bulk evals sync; live chat batch | Batch async vs sync interactive |
| 21 | Unbounded context | append forever | compact + externalize memory |

**Exam signal (meta):** When two options both look correct, prefer the one that is **deterministic, observable, least-privilege, and just-in-time**. The other is almost always the dressed-up anti-pattern.

---

## Sources

- Anthropic Docs — Messages API, tool use, prompt caching, extended thinking, Batch API, structured outputs: https://platform.claude.com/docs
- Claude Code documentation (CLAUDE.md, hooks, slash commands, plan mode): https://docs.anthropic.com/en/docs/claude-code
- Model Context Protocol specification and server design: https://modelcontextprotocol.io
- Anthropic Engineering — "Building effective agents," multi-agent research system, and context-engineering guidance: https://www.anthropic.com/engineering
