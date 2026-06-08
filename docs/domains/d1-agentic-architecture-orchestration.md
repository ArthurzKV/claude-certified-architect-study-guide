# Domain 1: Agentic Architecture & Orchestration (27%)

Domain 1 is the heaviest weighted domain on the Claude Certified Architect – Foundations (CCA-F) exam. It tests whether you can choose the *right* architecture for a task, orchestrate multiple agents correctly, and build agentic loops that terminate, recover, and escalate based on deterministic signals rather than vibes. Most D1 questions anchor to scenario **S1 (Customer Support Resolution Agent)** or **S3 (Multi-Agent Research System)**. The official grounding is Anthropic's "Building Effective Agents" (https://www.anthropic.com/research/building-effective-agents) and the Claude Agent SDK docs (https://platform.claude.com/docs).

## Workflows vs Agents and the "Simplest Architecture That Works" Meta-Rule

The single most important concept in Domain 1 is the distinction between **workflows** and **agents**, and the meta-rule that governs every architecture choice.

A **workflow** is a system where LLM calls and tools are orchestrated through **predefined code paths**. The control flow is written by you; the model fills in the steps. Workflows are predictable, cheap, low-latency, and easy to debug.

An **agent** is a system where the **model itself dynamically directs its own process** — it decides which tools to call, in what order, and when it is finished, looping until a goal is met. Agents are flexible and handle open-ended problems, but they cost more tokens, add latency, and are harder to make reliable.

**The meta-rule: use the simplest architecture that works, and only add agentic autonomy when the task genuinely requires it.** Start with a single LLM call plus retrieval/in-context examples. Escalate to a workflow only when one call is insufficient. Escalate to a full agent only when you cannot enumerate the steps in advance. Anthropic explicitly recommends *not* reaching for a multi-agent framework when a deterministic workflow would do — added autonomy trades latency and cost for capability, and you should only pay that price when it buys you something.

**Exam signal:** When a scenario describes a *fixed, known sequence of steps* (e.g., "first classify the ticket, then look up the account, then draft a reply"), the correct answer is almost always a **workflow**, not an autonomous agent. A distractor will offer a flashy multi-agent design for a task that a prompt chain handles. Pick the simpler one.

**Exam signal:** Watch for the phrase "the model decides." If the task requires the model to choose *which* and *how many* steps at runtime (open-ended research, variable-depth debugging), that signals a true **agent**. If the steps are enumerable up front, that signals a **workflow**.

## The Five Workflow Patterns and Selection Criteria

You must know all five workflow patterns from "Building Effective Agents," what each is good for, and how to tell them apart in a scenario.

### Prompt Chaining (Sequential Decomposition)

Decompose a task into a fixed sequence of steps, where each LLM call processes the output of the previous one. You can add programmatic "gate" checks between steps to catch failures early. **Use when** the task cleanly splits into predictable subtasks and you want to trade latency for higher accuracy per step (each call does something simpler). Example: generate marketing copy → translate it → validate tone. Trap: do not chain when steps are independent — that wastes latency you could recover with parallelization.

### Routing (Classify-Then-Dispatch)

Classify an input and direct it to a specialized followup prompt or model. **Use when** inputs fall into distinct categories that are better handled separately, and conflating them would hurt one category to help another. Example: route easy support questions to a cheap/fast model (Haiku) and hard ones to a stronger model (Opus/Sonnet); route refund requests vs. technical questions to different prompts. This is the canonical **cost-optimization** pattern — route by difficulty to control spend.

### Parallelization (Sectioning and Voting)

Run multiple LLM calls simultaneously and aggregate. Two flavors: **sectioning** (split a task into independent subtasks run in parallel, e.g., review code for security, performance, and style separately) and **voting** (run the same task multiple times for diverse outputs or majority consensus, e.g., multiple independent checks for whether content is harmful). **Use when** subtasks are independent (sectioning) or when multiple attempts improve confidence (voting). Benefit: lower wall-clock latency and/or higher reliability through redundancy.

### Orchestrator-Workers (Dynamic Decomposition)

A central **orchestrator** LLM dynamically breaks a task into subtasks at runtime, delegates each to worker LLMs, and synthesizes their results. **Key difference from parallelization:** the subtasks are **not predefined** — the orchestrator decides them based on the input. **Use when** you cannot predict the subtasks in advance (e.g., a coding change that touches an unknown set of files, or a research question requiring an unknown number of sub-investigations). This is the workflow that most resembles a multi-agent system and underpins scenario S3.

### Evaluator-Optimizer (Generate-Critique-Refine)

One LLM generates a response; a second LLM **evaluates it against criteria and gives feedback** in a loop until the output passes. **Use when** you have clear evaluation criteria *and* iterative refinement measurably improves the result — e.g., literary translation, or complex search where an evaluator decides if more rounds are needed. The evaluator must be a **separate call with fresh context** so it does not inherit the author's bias (see anti-pattern #9).

**Exam signal:** Selection questions hinge on one discriminator. Independent subtasks → **parallelization**. Predefined sequence → **prompt chaining**. Categorize-then-specialize → **routing**. Subtasks unknown until runtime → **orchestrator-workers**. Clear rubric + iterative gains → **evaluator-optimizer**. Memorize these one-line discriminators.

## Hub-and-Spoke Multi-Agent Design (Coordinator/Subagent)

The exam's multi-agent model is **hub-and-spoke** (also called coordinator-subagent or orchestrator-worker), grounded in Anthropic's "How we built our multi-agent research system" (https://www.anthropic.com/engineering/built-multi-agent-research-system). A central **coordinator/orchestrator** agent owns the goal; **subagents** do the focused work.

The orchestrator's job is a three-phase loop: **decompose → delegate → synthesize.** It (1) decomposes the high-level objective into well-scoped subtasks, (2) delegates each to a subagent with a clear, self-contained instruction (objective, output format, boundaries, and which tools to use), and (3) synthesizes the subagents' returned results into a final answer.

**The orchestrator does NOT execute everything itself.** Its value is coordination, not doing all the tool calls in one bloated context. A classic wrong answer makes the coordinator personally run every search and tool call — that defeats the purpose, blows the context window, and serializes work that should run in parallel.

**Why hub-and-spoke wins for research-type tasks:** subagents run in **parallel** (latency win) and each operates in its **own isolated context window** (it can explore deeply without polluting the orchestrator's context). Anthropic reported that a multi-agent research system substantially outperformed a single agent on breadth-first queries, largely because parallel subagents collectively apply far more effective context to the problem.

**Exam signal (S3):** When a research/analysis scenario asks how to scale to many sources or sub-questions, the right answer is a coordinator that **spawns parallel subagents with isolated contexts** and synthesizes their findings. Distractors: (a) one mega-agent with 18 tools doing it all sequentially, or (b) a coordinator that does the work itself instead of delegating.

## Subagent Context Isolation

Each subagent should run in its **own context window**, receiving only the instruction and inputs it needs, and returning only a **condensed result** to the orchestrator — not its entire raw transcript. This is *context isolation*, and it is a core reliability technique.

Why it matters: it prevents context-window exhaustion in the orchestrator, keeps each subagent focused (less distraction, higher accuracy), and stops one subagent's intermediate noise from corrupting another's reasoning. The orchestrator's context stays clean — it sees curated summaries, not raw tool dumps. This is the multi-agent analogue of the single-agent practice of compacting/summarizing long histories.

**Exam signal:** If a scenario reports the orchestrator running out of context or degrading as subtask count grows, the fix is **better context isolation and result summarization at the subagent boundary**, not a bigger model or a higher token limit.

## The Agentic Loop and Programmatic Termination via stop_reason

An agent runs a loop: the model returns a response that may contain **tool_use** blocks; your harness executes those tools, appends the **tool_result** blocks to the conversation, and calls the model again. The loop continues as long as the model keeps requesting tools.

**Termination is determined programmatically by the API `stop_reason`, never by parsing the model's prose.** The loop continues while `stop_reason` is `tool_use` (the model wants to call a tool). The loop *ends* when `stop_reason` is `end_turn` (the model is done and produced a final answer). Other values like `max_tokens` or `stop_sequence` also end the turn and must be handled. Your control flow keys off this field — full stop.

**Anti-pattern (the single most-tested D1 trap): parsing natural language like "I'm done" or "Task complete" to decide when to stop.** Models phrase completion inconsistently, can say "done" mid-task, and can be prompt-injected into claiming completion. Always branch on `stop_reason`. The model's text is *content*, not a *control signal*.

**Iteration caps are a safety backstop, not the primary stopping mechanism.** A `max_iterations` ceiling exists to prevent runaway loops and cost blowups (a circuit breaker), but it must never be the *intended* way the agent finishes its work. If your agent regularly hits the cap to stop, your design is broken — the model should reach a natural `end_turn`. Hitting the cap should be treated as an anomaly worth logging or escalating, not the happy path.

**Exam signal:** Any answer that says "stop when the model says it's finished" or "the agent halts after N iterations" as the *primary* mechanism is wrong. The correct answer checks `stop_reason == tool_use` to continue and `end_turn` to stop, with the iteration cap framed explicitly as a backstop.

## Fault Tolerance: Retry with Exponential Backoff and State Preservation

Production agents must survive transient failures: rate limits (HTTP 429), overloaded responses (529), timeouts, and flaky tools. The standard pattern is **retry with exponential backoff and jitter** — wait progressively longer between attempts (e.g., 1s, 2s, 4s…) with randomization to avoid thundering-herd retries. Distinguish retryable errors (429/500/503/529, network timeouts) from non-retryable ones (400 bad request, 401 auth) — only retry the former.

**State preservation enables resumption.** Because agent runs are long and expensive, persist the conversation/loop state (message history, completed subtasks, intermediate results) durably so that after a crash or a process restart you can **resume from the last good checkpoint** instead of restarting from scratch. For very large fan-out, the **Message Batches API** (https://platform.claude.com/docs) gives you asynchronous, fault-tolerant processing of independent requests at lower cost — relevant when a scenario emphasizes throughput over latency.

**Exam signal:** A scenario where "a subagent failed halfway through a 40-minute research run" should pick **retry-with-backoff plus resume-from-saved-state**, not "restart the whole job" and not "swallow the error and return partial results as if complete" (anti-pattern #7).

## When to Escalate to a Human

Escalation decisions must be grounded in **measured task complexity and tool-failure signals**, not in soft proxies. The right triggers are deterministic and observable: repeated tool failures after retries, hitting the iteration backstop, the action being high-stakes/irreversible (refunds above a threshold, account deletion), required tools/permissions being unavailable, or low retrieval coverage for the question. These are things your *code* can detect.

**Anti-pattern #4 — self-reported confidence:** Do not escalate (or auto-resolve) based on a confidence number the model produces about itself. LLM self-confidence is poorly calibrated and easily gamed; "I'm 95% sure" is not evidence.

**Anti-pattern #5 — sentiment-based escalation:** Do not escalate because the customer sounds angry. **Customer sentiment ≠ task complexity.** A furious customer may have a trivially resolvable problem; a polite customer may have a deeply complex one. Sentiment can inform *tone*, never the *routing/escalation* decision. Tie escalation to the difficulty and risk of the task, enforced in **deterministic code/hooks**, not pleaded for in the prompt (anti-pattern #3).

**Exam signal (S1):** The Customer Support scenario loves escalation questions. The correct trigger is something like "escalate when the requested refund exceeds the policy threshold" or "escalate after N failed tool attempts" — enforced in a code hook. Wrong answers escalate on sentiment, on the model's self-confidence, or by asking the prompt nicely to "please escalate if unsure."

## Cost, Latency, and Reliability Trade-offs

Every architecture choice is a trade-off across three axes:

- **Cost** rises with autonomy and parallelism. Control it with **routing** (cheap model for easy inputs), **prompt caching** (reuse stable system prompts/context), and the **Batch API** (lower per-token cost for non-urgent bulk work).
- **Latency** falls with **parallelization** (sectioning, parallel subagents) and rises with long sequential chains and deep agentic loops.
- **Reliability** improves with deterministic control (stop_reason, hooks, validation-retry), evaluator-optimizer loops, voting/redundancy, and context isolation — and degrades with too many tools, polluted context, and prose-based control flow.

The meta-rule reappears here: don't pay autonomy's cost/latency tax unless the task's open-endedness requires it.

**Exam signal:** Trade-off questions test whether you can name the *specific lever*: "reduce cost without losing quality on a high-volume support queue" → **route easy tickets to a smaller model + cache the system prompt**; "cut latency on a 12-source research task" → **parallel subagents**; "this bulk overnight job is too expensive" → **Batch API**.

## Mapping to Scenarios S1 and S3

**S1 — Customer Support Resolution Agent.** Built on the Agent SDK with MCP tools. D1 surfaces here as: an agentic loop terminated by `stop_reason`; escalation gated by deterministic policy thresholds and tool-failure counts (never sentiment or self-confidence); a tight, curated toolset (~4–7 tools, not 18); and graceful error handling that surfaces diagnostic context to the model rather than hiding it.

**S3 — Multi-Agent Research System.** The canonical hub-and-spoke scenario. D1 surfaces as: a coordinator that decomposes → delegates to parallel subagents → synthesizes (and does NOT do all the work itself); subagent context isolation with summarized returns; orchestrator-workers when subtasks are unknown until runtime; and fault tolerance via retry-with-backoff plus resumable state when a long run partially fails.

## Top Traps for D1 (Memorize These Wrong Answers)

1. **Parsing "I'm done"** (or any natural language) to terminate the loop instead of checking `stop_reason`. (#1)
2. **Iteration caps as the primary stop mechanism** — they are only a safety backstop; the model should reach a natural `end_turn`. (#2)
3. **Prompt-based enforcement of critical business rules** (e.g., refund limits) instead of deterministic code/hooks. (#3)
4. **Self-reported confidence scores driving escalation** — model self-confidence is uncalibrated. (#4)
5. **Sentiment-based escalation** — customer mood is not task complexity. (#5)
6. **Generic error messages** that strip diagnostic context the model needs to recover. (#6)
7. **Silently suppressing errors** — returning empty/partial results as if the task succeeded. (#7)
8. **Too many tools on one agent (e.g., 18)** instead of a curated ~4–7 set split across subagents/MCP boundaries. (#8)
9. **Same-session self-review** — the evaluator inherits the author's bias; use a fresh-context evaluator agent. (#9)
10. **The orchestrator executing everything itself** instead of delegating to subagents — defeats hub-and-spoke, exhausts context, serializes parallelizable work.

## D1 Quick-Reference Decision Table

- Steps known and fixed → **Workflow** (chain/route/parallelize).
- Steps unknown until runtime, model must decide → **Agent / orchestrator-workers**.
- Independent subtasks → **Parallelization (sectioning)**; same task repeated for consensus → **voting**.
- Categorize then specialize → **Routing** (also the cost lever).
- Clear rubric + iterative gains → **Evaluator-optimizer** (fresh-context evaluator).
- Many sub-investigations at scale → **Hub-and-spoke**, parallel subagents, isolated contexts, coordinator synthesizes.
- Loop control → **`stop_reason`** (`tool_use` continue, `end_turn` stop); cap is a **backstop**.
- Failures → **retry + exponential backoff + jitter**, **persist state**, **resume**; bulk/cheap → **Batch API**.
- Escalation → **measured complexity / tool-failure / risk thresholds in code**, never sentiment or self-confidence.
