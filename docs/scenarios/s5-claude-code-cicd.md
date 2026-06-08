# Scenario 5: Claude Code for CI/CD — Automated Multi-Pass Review Pipeline

## Scenario Overview: Claude Code as a Non-Interactive CI/CD Gate

S5 puts Claude Code inside a continuous-integration pipeline (GitHub Actions, GitLab CI, Jenkins, Buildkite) where it reviews pull requests, audits diffs, scans many files for security or style issues, and **gates** the build by producing a machine-readable verdict. There is no human at a terminal. The architectural challenge is reliability and determinism: the pipeline step must emit structured output a downstream job can parse, fail the build on real problems, and stay cheap and fast enough to run on every PR.

Exam signal: S5 questions almost always frame Claude as a *step in an automated pipeline* whose output is consumed by another program (a gate job, a status check, a comment bot). The moment a stem says "the next pipeline step needs to decide pass/fail," the right answer involves **structured JSON output**, not free text.

## Headless Mode: `claude -p` for Non-Interactive Pipelines

The foundation of S5 is **headless mode**, invoked with `claude -p "<prompt>"` (also written `--print`). This runs Claude Code once, non-interactively, prints the result to stdout, and exits — perfect for a CI job that has no TTY. You pipe the prompt or repo context in, capture stdout, and act on the exit code or parsed payload.

Key flags that show up in correct answers: `--output-format json` (or `stream-json`) to get a structured envelope instead of prose; `--allowedTools` / `--disallowedTools` to constrain what Claude can do in the sandbox; `--permission-mode` to control whether actions need approval (in CI you cannot answer prompts, so you pre-authorize a tight tool set rather than running fully open). The official reference is the Claude Code headless/SDK docs at https://docs.anthropic.com/en/docs/claude-code/.

Exam signal: If an option suggests running the *interactive* REPL in CI, or relying on a human to approve a permission prompt mid-pipeline, it is wrong — CI is non-interactive. The right answer is `claude -p` with `--output-format json` and an explicit allowed-tools list.

## Producing Structured Output for Machine Consumption (Gate the Build)

A CI gate must branch on a value, not read an essay. The correct pattern is to have Claude emit **structured JSON** that the next step parses with `jq` (or a script) to decide pass/fail. Two complementary mechanisms:

1. **Claude Code `--output-format json`** wraps the run in a JSON envelope (result text, cost, session metadata). This is the transport layer for the pipeline.
2. **Tool-use / output schema for the verdict itself.** To guarantee the *content* is well-formed, define a tool the model must call (e.g. `submit_review`) whose `input_schema` is the JSON Schema of your verdict — fields like `verdict` (`pass`/`fail`/`needs_changes`), `severity`, `findings[]` with `file`, `line`, `category`, `message`. Because the model populates a typed `tool_use` block, you get schema-shaped JSON instead of hoping prose parses. This is the structured-outputs/tool-use approach documented at https://platform.claude.com/docs (Messages API, tool use).

The downstream gate then does: parse JSON → if `verdict == "fail"` or any finding `severity >= high`, exit non-zero and fail the build; otherwise pass. The decision is deterministic and lives in *code*, not in the model's good intentions.

Exam signal: The classic WRONG answer is "have Claude write a review and grep the text for the word 'LGTM' / 'approved' / 'pass'." Parsing free-text natural language to gate a build is brittle and is the CI analog of anti-pattern #1 (parsing "I'm done" instead of a structured signal). Always prefer a typed `tool_use` verdict or an output schema.

### Why Not Free-Text Parsing

Free text drifts: the model phrases the same verdict ten ways, embeds caveats, or buries the decision in a paragraph. A regex over prose is a hidden failure waiting to happen — a reworded sentence silently flips the gate. Schema-constrained output removes the ambiguity: the field is either present and valid or the run is retried/failed loudly. This connects to D4 (structured output) and to anti-pattern #7 (don't let a malformed result pass as success).

## Batch API vs Synchronous: Latency-Sensitive PR Feedback vs Bulk Review

S5 forces a throughput-vs-latency decision, and the exam tests whether you pick the right execution mode.

**Synchronous Messages API (and a normal `claude -p` run)** — use when a human is waiting. Fast PR feedback, a single changed file, a blocking required status check that must return in seconds-to-minutes. Real-time, pay standard rate.

**Batch API** — use for non-latency-sensitive bulk work: auditing an entire repository, reviewing hundreds of files or a giant migration diff, nightly scheduled scans, backfilling review on a large monorepo. The Batch API processes requests **asynchronously** (results within ~24 hours, often much sooner) at roughly **50% lower cost** than synchronous calls. The trade-off is you cannot block a PR on it — you submit, poll, and consume results later. Reference: Message Batches at https://platform.claude.com/docs.

Decision rule: *Is a human or a required check waiting on this result right now?* If yes → synchronous. If no (overnight audit, mass file review, cost-sensitive bulk job) → Batch API for the ~50% savings and higher throughput.

Exam signal: Picking the **synchronous** API for a huge, non-urgent bulk job (e.g. "review all 4,000 files in the repo for license headers every night") is a WRONG answer — it's slower at scale and wastes ~50% of the budget. Conversely, using the **Batch API** to gate a live PR that needs an answer in 60 seconds is also wrong because batch is async (~24h SLA). Match mode to urgency.

## Multi-Pass Review with a FRESH Context per Pass

High-quality CI review uses **multiple specialized passes** — e.g. pass 1 security, pass 2 correctness/logic, pass 3 style/maintainability, pass 4 test coverage. The critical architectural rule: **each pass runs in a fresh context (a new session / new agent), and the reviewer is never the same context that authored or already analyzed the code.**

Why fresh context matters: a reviewer that shares the session with the author (or with a prior pass) inherits that reasoning chain and rationalizes the same blind spots — it "agrees with itself." This is **anti-pattern #9: same-session self-review.** A clean context re-derives judgment from the artifact alone, which is exactly what an independent reviewer should do. In practice you spawn a separate `claude -p` invocation (or subagent) per pass, each with a focused system prompt and only the inputs it needs.

Exam signal: When a stem describes "Claude generates the code, then in the same conversation we ask Claude to review its own work," the correct critique is that the reviewer inherits the author's bias — use a **separate, fresh-context review pass**. Any option that keeps generation and review in one session is the trap.

### Specialized Passes and Result Merging

Each pass emits its own structured findings (same schema, tagged with `pass`/`category`). A deterministic merge step in code aggregates them, dedupes, and computes the final gate decision. Splitting concerns across passes also keeps each agent's tool set and prompt tight (D2: ~4–7 tools, not one mega-reviewer with 18 — anti-pattern #8) and makes each pass independently cacheable and debuggable.

## Hooks and Permissions: Enforce Gates Deterministically

The pass/fail gate must be enforced in **code/config, not by asking the model nicely.** Claude Code **hooks** (`PreToolUse`, `PostToolUse`, `Stop`, etc., configured in `settings.json`) and the **permission system** (`--allowedTools`, `--permission-mode`, `settings.json` permission rules) let the pipeline guarantee behavior regardless of what the model decides. Example: a `PreToolUse` hook blocks any write outside the workspace; a wrapper script enforces "exit non-zero if any finding is severity high" in the CI YAML, not in the prompt.

This is **anti-pattern #3** applied to CI: prompt-based enforcement ("please fail the build if you find a critical bug") is non-deterministic. Deterministic enforcement lives in hooks, permission rules, and the gate script. The model *reports*; your code *decides and enforces*. Hooks reference: https://docs.anthropic.com/en/docs/claude-code/.

Exam signal: If the only thing stopping a bad merge is a sentence in the prompt, that's the wrong design. The right design puts the gate in a hook or in the pipeline's parse-and-exit logic — deterministic, testable, model-independent.

## Per-File / Per-Category Metrics, Not One Aggregate Score

A single aggregate score ("repo scored 92/100") **masks localized failures**: one file with a critical SQL-injection finding can be drowned out by 200 clean files. The correct CI design reports **per-file and per-category** results — findings keyed by file, line, and category (security, performance, correctness, style) — and gates on the *worst slice*, not the mean.

This is **anti-pattern #10**: aggregate accuracy hides per-slice failure. In CI it's worse than misleading — it can let a critical vulnerability merge because the average looked fine. Surface every finding with its location and severity; let the gate fail on any high-severity item regardless of the overall average.

Exam signal: An option offering "compute one overall quality score and merge if it's above the threshold" is the trap. The right answer breaks results down per file/category and fails on the most severe finding.

## Prompt Caching for Repeated Repo Context Across Files

When reviewing many files, the same large context repeats every call: the `CLAUDE.md`/system prompt, coding standards, the security policy, shared schema/types, architectural docs. **Prompt caching** stores that stable prefix so subsequent requests reuse it at a large discount (cache reads are far cheaper than fresh input tokens) and lower latency. Put the *invariant* context (standards, policy, repo conventions) in the cached prefix and vary only the per-file diff after it. Reference: prompt caching at https://platform.claude.com/docs.

Cache hygiene matters: the cached prefix must be byte-stable and placed *before* the variable content, since caching keys on a matching prefix. Reordering or editing the prefix per file destroys the hit. Combined with the Batch API, caching makes large repo-wide reviews dramatically cheaper.

Exam signal: For "review 500 files that all share the same coding-standard context," the cost/latency win is **prompt caching the shared prefix** (and Batch API for the bulk). An answer that re-sends the full standards doc fresh on every file ignores caching and is wrong on cost.

## CI/CD Decision Tree

1. **Is a human / required status check waiting on this result?**
   - Yes → synchronous `claude -p` with `--output-format json`. Fast PR feedback.
   - No (nightly audit, mass-file scan, cost-sensitive bulk) → **Batch API** (~50% cheaper, async ~24h).
2. **How does the next step consume the result?**
   - It must branch on it → **structured JSON via tool_use / output schema**, parsed with `jq`. Never grep free text.
3. **Is the review one pass or many?**
   - Multiple concerns (security, correctness, style, tests) → **separate fresh-context passes**, never same-session self-review.
4. **How is the gate enforced?**
   - In **hooks / permission rules / parse-and-exit script** (deterministic) — never via a plea in the prompt.
5. **How are results reported?**
   - **Per-file / per-category findings**, gate on worst severity — never a single aggregate score.
6. **Repeated large context across files?**
   - **Prompt-cache the stable prefix** (standards, policy, CLAUDE.md); vary only the per-file diff.

## Anti-Pattern → Right-Pattern Quick Map (S5)

- Grep prose for "approved"/"LGTM" to gate → **typed `tool_use` verdict / output schema**, branch in code (relates to anti-pattern #1).
- Generate and review code in the **same session** → **fresh context / separate review pass per concern** (anti-pattern #9).
- One **aggregate score** decides the merge → **per-file/per-category findings**, fail on worst slice (anti-pattern #10).
- **Synchronous** API for a huge non-urgent bulk job → **Batch API** (~50% cheaper, async).
- **Batch API** for a live PR needing an answer in seconds → **synchronous** `claude -p`.
- Enforce the gate by **asking the model in the prompt** → **hooks + permission rules + parse-and-exit** (anti-pattern #3).
- Re-send shared standards/context **fresh per file** → **prompt-cache the stable prefix**.
- Suppress a parse/tool error and **pass the build anyway** → fail loudly and retry; never hide errors (anti-pattern #7).
- One **18-tool mega-reviewer** → split concerns into focused passes/subagents with ~4–7 tools each (anti-pattern #8).

## Cross-Domain Mapping

S5 is mostly **D3 (Claude Code config/workflows: headless mode, hooks, permissions)** and **D4 (structured output for machine consumption)**, with strong **D5 (reliability: deterministic gates, fail-loud, per-slice metrics)** and a **D1 (orchestration: multi-pass fresh-context review)** component. Batch vs sync and prompt caching are Claude API knowledge (Messages, batch, caching) applied to the pipeline. Expect stems that combine two or more of these — e.g. "cheap nightly repo-wide security review that gates the build per file" maps to Batch API + tool_use schema + per-file findings + prompt-cached standards all at once.
