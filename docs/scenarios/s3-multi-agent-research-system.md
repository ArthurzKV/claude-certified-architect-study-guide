# Scenario 3 — Multi-Agent Research System (CCA-F Deep Dive)

## Scenario 3 Overview: Coordinator and Worker-Subagent Research Architecture

Scenario 3 (S3) tests whether you can design a **multi-agent research system** that decomposes a broad, open-ended question into independent sub-questions, runs **parallel subagents** to investigate each, and synthesizes their condensed findings into a single cited answer. The reference architecture is the **orchestrator-worker** (coordinator/subagent) pattern: one lead agent plans and delegates; multiple worker subagents execute in isolated context windows and return compressed results. This mirrors Anthropic's own engineering write-up "How we built our multi-agent research system" (https://www.anthropic.com/engineering/built-multi-agent-research-system) and the Claude Agent SDK subagent model (https://docs.anthropic.com/en/api/agent-sdk).

Exam signal: S3 questions hand you a research/analysis workload and ask for the *control structure*. The right answer almost always describes **decompose → delegate (parallel) → synthesize** with isolated subagent contexts. Watch for distractors that propose a single mega-context agent, prompt-based termination, or provider-switching mid-task.

## The Core Pattern: Decompose, Delegate, Synthesize

The orchestrator's job is **planning and aggregation**, not doing the research itself. A clean S3 design has three phases:

1. **Decompose** — the lead agent reads the user's research question and breaks it into independent, non-overlapping sub-questions. Good decomposition produces sub-questions that can be answered *without* each other's results (true parallelism). If sub-question B needs B-answers-from-A, those are sequential, not parallel, and you must say so.

2. **Delegate** — the orchestrator spawns one subagent per independent sub-question. Each subagent gets a **clean, isolated context window** containing only its task description and the tools it needs (search, fetch, read). It does NOT inherit the orchestrator's full transcript.

3. **Synthesize** — each subagent returns a **condensed finding** (a short summary plus the specific sources/citations it relied on), not its raw scratch work. The orchestrator aggregates these condensed findings and composes the final answer with citations. The orchestrator never sees the subagents' intermediate tool spew, which is the whole point.

Exam signal: if an answer choice has the lead agent personally running every web search and reading every page in one window, it is wrong — that is the single mega-context anti-pattern. The lead should *orchestrate*, and workers should *do*.

## Parallel Subagents for Independent Sub-Questions

The performance win of multi-agent research comes from **fan-out**: launching independent subagents concurrently so several lines of inquiry advance at once, then aggregating. Anthropic reported that the multi-agent research system substantially outperformed a single agent on breadth-first research tasks, largely because parallel subagents cover more ground and because each subagent uses its own context budget.

Parallelism is only valid when sub-questions are **genuinely independent**. If question 2 depends on question 1's output, you sequence them or have one subagent do both. Mislabeling dependent work as parallel is a trap. The decomposition step is where you decide this — the orchestrator should explicitly classify sub-questions as independent (fan out) vs. dependent (chain).

Exam signal: "Which sub-questions can run in parallel?" The correct reasoning is dependency-based, not count-based. Independent → parallel; dependent → sequential. An answer that parallelizes everything blindly is wrong; so is one that serializes everything "to be safe."

## Subagent Context Isolation: Fighting Context Limits

The strategic reason multi-agent helps is **context isolation**. Each subagent operates in its own window, so the noisy, token-heavy work of reading dozens of pages happens *inside* the subagent and never pollutes the orchestrator's context. The subagent distills its findings and returns only the distilled version. This is effectively a divide-and-conquer strategy for the context window: total useful work scales with the number of subagents, while the orchestrator's context stays small and focused on planning + synthesis.

This is why "just use one agent with a huge context window" is the wrong answer even when a large window is technically available. A single window degrades on long-horizon, many-source tasks (attention dilution, lost-in-the-middle, runaway token cost), and one error early can poison the entire remaining trajectory. Isolation contains failures and keeps each reasoning trace tight.

Exam signal: the phrase "each subagent returns a condensed finding / summary, not its full transcript" is the marker of the correct answer. Distractors return raw transcripts to the orchestrator (defeats isolation) or share one context across all workers (defeats the point).

## Error Handling: Retry With Backoff, Preserve State, Don't Tear Down

When a subagent hits a **transient failure** (a flaky tool call, a rate limit, a timeout, a 5xx from a search/MCP tool), the correct response is to **retry that subagent with exponential backoff while preserving the rest of the pipeline's state**. You do NOT tear down the whole run because one worker stumbled, and you do NOT discard the work the other subagents already completed.

Concretely: catch the error at the orchestration layer, retry the single failing subagent (bounded retries with backoff/jitter), and keep the successful subagents' condensed findings intact in orchestrator state. If a subagent still fails after retries, the orchestrator should synthesize with what it has and **explicitly flag the gap** in the final answer — not silently drop it.

The model must NOT be asked to "self-diagnose via prompt" — i.e., you don't paste a generic "something went wrong, figure it out" string into the model and hope it recovers. Retry/backoff is **deterministic orchestration code**, a control-plane concern, not a reasoning task. Likewise, **error messages returned to the model should preserve diagnostic context** (what tool, what input, what error) so the model can adapt if a retry is genuinely the right call — never a generic opaque message, and never a silently-suppressed error that returns empty results as if successful.

Exam signal: maps to anti-patterns #6 (generic error messages hide context) and #7 (silent error suppression). The right answer retries the failing unit with backoff, preserves partial progress, and surfaces unresolved gaps honestly.

## Termination: stop_reason and State, Not Natural Language

Each subagent's loop terminates by inspecting the **API `stop_reason`** and the orchestrator's tracked **state** — not by parsing natural-language phrases like "I'm done" or "research complete." When a subagent's response comes back with `stop_reason: "end_turn"` (no further tool calls requested), that turn's work is finished; `stop_reason: "tool_use"` means run the tool and feed results back. The orchestrator decides the *overall* run is complete when all delegated sub-questions have returned findings (a state check), not when the model emits a done-sounding sentence.

Iteration caps exist only as a **safety backstop** against runaway loops — they are not the primary stopping mechanism. The primary control is: stop when `stop_reason` indicates completion AND orchestrator state shows all sub-questions resolved (or exhausted retries).

Exam signal: maps to anti-patterns #1 (parsing NL for termination) and #2 (iteration cap as primary control). Correct termination = `stop_reason` + completion state. Any answer keying off the model "saying" it's finished is a distractor.

## Verification and Citation of Findings

Research output is only as trustworthy as its sources. Subagents must **attach citations** (the specific URLs/documents that support each claim) to their condensed findings, and the orchestrator should preserve and surface those citations in the final answer. Where correctness matters, add a **verification pass** — ideally a *separate* agent / fresh context that checks claims against the cited sources rather than the same agent re-grading its own work.

Same-session self-review is an anti-pattern (#9): a reviewer that inherits the author's reasoning inherits its blind spots. A clean-context verifier (or a dedicated citation-checking subagent) catches unsupported claims and hallucinated sources that the original author rationalizes.

Exam signal: "How do you make the research trustworthy?" → cite sources per finding + verify with a fresh-context reviewer. Distractor: "ask the same agent to double-check its answer."

## Cost and Latency Trade-Offs of Fan-Out

Multi-agent research is **token- and cost-intensive**: each subagent runs its own model loop with its own tool calls, so a fan-out of N subagents plus an orchestrator can consume many times the tokens of a single agent. Anthropic noted multi-agent systems can burn on the order of ~15× the tokens of a single chat for the same wall-clock task. That cost buys breadth, parallel latency reduction, and context isolation — but it is only justified when the task genuinely benefits from parallel decomposition.

Latency is reduced by parallelism (workers run concurrently) but each added subagent adds orchestration overhead and aggregation cost. Tune the **breadth of fan-out** to the task: enough subagents to cover independent sub-questions, not so many that you pay for redundant or trivially-decomposable work.

Exam signal: a question will trade cost vs. capability. The correct answer reserves multi-agent for high-value, breadth-first, parallelizable research and uses a single agent for narrow, sequential, or cheap tasks.

## When Multi-Agent Is Justified vs. a Single Agent

Use the **orchestrator-worker** pattern when ALL of the following hold: the task is open-ended/breadth-first; it decomposes into **genuinely independent** sub-questions; parallel exploration meaningfully improves quality or latency; and the per-subagent context isolation is worth the extra token cost. Research, broad codebase exploration, and multi-source synthesis fit well.

Use a **single agent** when the task is narrow, mostly sequential, has tight cross-step dependencies, or is cost-sensitive and small. Forcing multi-agent onto a linear task just adds coordination overhead, token cost, and failure surface with no parallelism benefit. Don't multiply agents for prestige.

## Decision Tree: Single Agent vs. Multi-Agent Research

```
Is the task open-ended / breadth-first research or broad exploration?
├─ No  → Single agent. (Multi-agent adds cost & coordination for no gain.)
└─ Yes → Can it decompose into INDEPENDENT sub-questions?
         ├─ No (tight dependencies) → Single agent OR sequential chain. Not parallel fan-out.
         └─ Yes → Does parallel exploration improve quality/latency enough to justify ~Nx token cost?
                  ├─ No  → Single agent.
                  └─ Yes → ORCHESTRATOR-WORKER:
                           1. Lead decomposes → independent sub-questions
                           2. Spawn 1 subagent per sub-question (PARALLEL), isolated context
                           3. Each subagent returns CONDENSED finding + citations
                           4. On transient subagent failure → retry w/ backoff, preserve state
                           5. Terminate per subagent via stop_reason; overall via completion state
                           6. Orchestrator synthesizes; fresh-context verifier checks citations
                           7. Surface unresolved gaps honestly
```

## Likely WRONG Answers (S3 Distractors) and Why

- **Single mega-context agent for everything.** Defeats context isolation; degrades on long-horizon multi-source work; one early error poisons the whole trajectory. Wrong even with a large context window.

- **Prompt-based / natural-language termination** ("loop until the agent says 'done'"). Violates anti-pattern #1. Use `stop_reason` + completion state.

- **Iteration cap as the primary stopping mechanism.** Anti-pattern #2. Caps are a safety backstop only.

- **Provider-switching / model-swapping mid-task** to recover from a transient error. Loses state and consistency; the fix for a transient failure is retry-with-backoff on the same unit, not switching providers mid-flight.

- **No state preservation** — tearing down the whole pipeline (or restarting from scratch) when one subagent fails. Discards completed work; correct design preserves partial progress and retries only the failed unit.

- **Subagents return raw transcripts** to the orchestrator. Defeats context isolation and blows the orchestrator's context budget.

- **One shared context across all subagents.** Same problem as the mega-agent; no isolation, no real fan-out benefit.

- **Same-agent self-review** of findings. Anti-pattern #9; use a fresh-context verifier.

- **Generic/opaque error strings or silent suppression.** Anti-patterns #6 and #7; preserve diagnostic context and surface gaps.

- **Self-reported confidence driving whether to keep researching.** Anti-pattern #4; drive control flow off state and stop_reason, not the model's self-rated confidence.

## Anti-Pattern → S3 Mapping (Quick Reference)

| Anti-pattern | How it shows up in S3 | Correct pattern |
|---|---|---|
| #1 NL termination | "Loop until subagent says 'done'" | Check `stop_reason` + completion state |
| #2 Iteration cap as control | "Stop after 10 loops" as the mechanism | Cap = backstop; state drives stopping |
| #4 Confidence-driven control | "Keep going if confidence < 0.8" | State/stop_reason, not self-rated confidence |
| #6 Generic error messages | Opaque "an error occurred" to the model | Preserve tool/input/error diagnostic context |
| #7 Silent suppression | Failed subagent returns empty as success | Retry w/ backoff; flag unresolved gap |
| #8 Too many tools | Each subagent given 18 tools | Tight curated toolset per subagent (~4–7) |
| #9 Same-session self-review | Author agent grades its own findings | Fresh-context verifier checks citations |
| #10 Aggregate metrics mask slices | "Overall research looks good" | Verify per sub-question / per source |

Exam signal: S3 rewards the candidate who treats orchestration (decomposition, retry/backoff, termination, state) as **deterministic control-plane code** and treats the model as the worker that researches and synthesizes — never the other way around.
