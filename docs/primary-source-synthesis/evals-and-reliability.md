# Evaluation & Reliability for Claude-Based Systems (CCA-F)

This reference covers how to measure, verify, and harden agentic and structured-output systems built on Claude. It maps to **D5 Context Management & Reliability (15%)** and **D1 Agentic Architecture & Orchestration (27%)**, and surfaces most heavily in **S1 Customer Support Resolution Agent** and **S5 Claude Code for CI/CD**. Primary sources: the "Test & evaluate" section of the Claude docs (platform.claude.com/docs), Anthropic's engineering guidance on building effective agents (anthropic.com/research/building-effective-agents), and docs.anthropic.com/en/docs/claude-code.

## Build Evals BEFORE Optimizing Prompts (Empirical, Not Vibes)

The foundational reliability principle is that you cannot improve what you cannot measure. Anthropic's prompt-engineering guidance is explicit: define success criteria and build an evaluation harness **before** you start iterating on prompts. "Vibes-based" tuning — eyeballing a few outputs and declaring victory — produces prompts that look good on the examples you happened to read and regress silently on everything else.

The correct workflow is: (1) write down measurable success criteria, (2) assemble a representative test dataset, (3) implement graders, (4) establish a baseline score, then (5) iterate on the prompt or architecture and re-score against the same dataset. Every change is judged by movement on the eval, not by a single hand-picked example.

**Exam signal (D4/D5):** When a scenario describes a team "tweaking the prompt until the demo looked right" or shipping a prompt change "because the new output read better," the correct answer almost always involves establishing an evaluation set first and measuring the change empirically. Distractors will frame more prompt engineering or a bigger model as the fix; the root cause is the absence of a measurement loop.

## Golden Datasets and Representative Test Coverage

A **golden dataset** is a curated set of inputs paired with known-good expected outputs (or known-good acceptance criteria). It should mirror the real distribution of production traffic, including edge cases, adversarial inputs, and the "long tail" of rare-but-important categories. A dataset of only happy-path examples gives a falsely high score and hides exactly the failures that hurt in production.

Aim for enough examples per category that a single failure does not swing the score wildly. Keep the golden set version-controlled and treat it as a living asset: when a production incident reveals a missed case, add that case to the dataset so the regression can never silently return. This is the practical embodiment of "tests encode lessons learned."

**Exam signal (S6/S5):** Structured-extraction and CI scenarios reward datasets that include malformed inputs, missing fields, and ambiguous documents — not just clean samples. If an option says "test only on the documents that parsed successfully last quarter," it is wrong; you must include the inputs that previously failed.

## Code-Graded vs Model-Graded Evaluations: When Each Fits

There are two grader families, and choosing correctly is a recurring exam theme.

**Code-graded (programmatic) evals** use deterministic logic: exact string match, regex, **JSON-schema validation**, numeric tolerance, unit-test assertions, or "does this code compile and pass its tests." They are fast, free, perfectly reproducible, and unambiguous. Use them whenever the success criterion can be expressed in code. For S6 (structured data extraction) and S5 (CI/CD), code grading is the default: a JSON output either validates against the schema or it does not; extracted fields either match the ground truth or they do not.

**Model-graded evals (LLM-as-judge)** use a separate Claude call to score an output against a rubric. Use them only when correctness is genuinely subjective or open-ended — tone, helpfulness, faithfulness to a source, "did the summary capture the key points," whether a support reply is empathetic and on-policy. They are flexible but slower, costlier, and noisier than code graders, so reserve them for criteria code cannot express.

The decision rule: **prefer code grading; reach for model grading only when the criterion is inherently qualitative.** A frequent wrong answer is using an LLM judge to check something a schema validator or exact-match assertion would verify deterministically — that adds cost and nondeterminism for no benefit.

**Exam signal (D4):** "The output must be valid JSON conforming to this schema" → code-graded (schema validation), never an LLM judge. "The response should be polite and acknowledge the customer's issue" → model-graded rubric.

## Designing a Good LLM-as-Judge Rubric

When you do use a model grader, the rubric design determines whether the score means anything. Best practices:

**Give the judge clear, specific criteria.** Vague instructions like "rate the quality 1–10" produce noisy, uncalibrated scores. Decompose quality into concrete pass/fail or low-cardinality questions ("Does the answer cite the provided source? Yes/No", "Does it contradict any source fact? Yes/No"). Binary and ordinal rubrics are more reliable than open 1–100 scales.

**Make the judge independent of the author.** The grading prompt should evaluate the output against the rubric and the ground truth/source — it should not see the author's chain of thought or be the same in-context agent that produced the answer. An independent judge with fresh context avoids inheriting the author's assumptions.

**Avoid same-session self-grading bias (anti-pattern #9).** If the same agent in the same session both produces an answer and then "reviews" it, the review inherits the author's reasoning bias and tends to rubber-stamp its own work. Use a fresh context window or a separate agent for review/grading. This is the single most-tested reliability trap for multi-pass review.

**Exam signal (S5):** A multi-pass code review where the same session writes the code and then "reviews" it is the wrong design. The right design spins up a fresh-context reviewer (separate agent / new session) so the reviewer evaluates the diff without the author's rationalizations.

## Per-Category and Per-Document-Type Metrics (Not One Aggregate)

A single aggregate accuracy number is dangerous because it averages away localized failures (anti-pattern #10). "94% overall accuracy" can hide that one document type extracts at 40% while common types hit 99% and dominate the mean. The high-volume categories mask the broken low-volume ones, and you ship a system that is quietly useless for a whole class of inputs.

The fix is to **slice metrics by every dimension that matters**: per document type, per customer tier, per language, per intent category, per tool, per input length bucket. Report a breakdown, not a scalar. Set per-slice thresholds so a category dropping below its floor fails the eval even if the aggregate looks fine. This is how you catch the "minority category collapse" that aggregate metrics conceal.

**Exam signal (S6/S1):** When an option presents "overall accuracy is 92%, ship it" against "accuracy is 92% overall but only 55% on invoices," the per-category breakdown is the correct, more rigorous choice. Aggregate-only evaluation is a named anti-pattern and a reliable distractor.

## The "Verify Work" Step of the Agent Loop

A robust agent loop is gather context → take action → **verify work** → repeat. The verification step is where reliability lives: after the model acts (calls a tool, edits a file, produces output), the system checks the result against an objective signal before continuing. For coding agents that means running tests, compilers, linters, or type-checkers; for extraction it means schema validation; for support it means confirming the action actually succeeded.

Verification must come from **deterministic, external feedback**, not from the model's self-assessment. "I believe this is correct" is not verification; a passing test suite or a schema-valid payload is. Where possible, build the loop so the model receives the verification result (test output, validation error) and can self-correct — which leads directly to the validation-retry pattern.

**Validation-retry (S6):** When structured output fails schema validation, return the **specific validation error** to the model and let it retry, rather than discarding the result or returning empty. Generic error messages (anti-pattern #6) and silent suppression (anti-pattern #7) both rob the model of the diagnostic context it needs to fix the problem. Surface the real error.

**Exam signal (D1/D5):** Loop control and termination must key off the API **`stop_reason`** (e.g., `end_turn` vs `tool_use`), not natural-language cues like "I'm done" (anti-pattern #1). Iteration caps are a safety backstop, not the primary stopping mechanism (anti-pattern #2).

## Regression Suites in CI

Once you have a golden dataset and graders, wire them into continuous integration. Every prompt change, model upgrade, tool modification, or system-prompt edit should re-run the eval suite and block the merge if scores regress past a threshold. This converts reliability from a one-time exercise into a standing guarantee and is the mechanism that makes model upgrades safe — you can A/B a new model against the suite before promoting it.

For S5 (CI/CD with Claude Code), the **Batch API** is the natural fit for running a large eval set or multi-pass review asynchronously and cost-effectively, and **structured outputs / tool_use** make each result machine-checkable so the pipeline can pass/fail deterministically. Pin the dataset version and record scores over time so you can attribute regressions to specific changes.

**Exam signal (S5):** "Run the full eval set on every PR and fail the build if any per-category score drops below its floor" is the strong CI answer. Running evals manually and occasionally, or only spot-checking, are weak distractors.

## Tracing and Observability

You cannot debug what you cannot see. Instrument the system to capture, per request: the full message sequence, every tool call with its inputs and outputs, `stop_reason`, token usage, latency, retries, and the final result. Structured traces let you reconstruct exactly why an agent took a wrong turn, find which tool returned bad data, and identify the slice where quality is dropping.

Observability also feeds evaluation: production traces are the richest source of new golden-dataset cases and reveal the real input distribution your offline set should mirror. Log errors with full diagnostic context — never swallow them — so failures are visible and attributable rather than hidden behind empty results (anti-pattern #7).

**Exam signal (D5):** When a scenario asks how to diagnose intermittent agent failures in production, the answer involves structured tracing/observability of tool calls and stop reasons — not adding more prompt instructions or simply increasing the retry count blindly.

## Escalation Thresholds Grounded in Measured Task Complexity

Escalation (handing off to a human or a more capable model/path) must be triggered by **objective, measured signals of task complexity or failure**, not by proxies that don't track correctness.

Wrong triggers, all named anti-patterns: the model's **self-reported confidence score** (anti-pattern #4 — models are poorly calibrated and will assert confidence on wrong answers); **customer sentiment** (anti-pattern #5 — an angry customer may have a trivial request, and a polite one may have an impossible one; sentiment ≠ complexity); and pleading prompt instructions to "escalate if unsure" (anti-pattern #3 — prompt-based enforcement of a critical business rule).

Right triggers: deterministic conditions measured in code — a required tool returned an error, schema validation failed after N retries, the request matches a known out-of-scope category, a required field could not be resolved, the policy engine flagged the action, or a measured task-complexity classifier crossed a calibrated threshold. Critical escalation rules belong in **deterministic hooks / code**, not in the prompt.

**Exam signal (S1):** A support agent that escalates based on "low confidence" or "negative sentiment" is the trap. The correct design escalates on measured, deterministic conditions (tool failure, validation failure, out-of-scope classification, policy violation) enforced in code/hooks rather than asked for in the prompt.

## Quick Reference: Reliability Anti-Patterns to Recognize on the Exam

Pair each wrong pattern with its fix: parse `stop_reason`, not "I'm done" (#1); iteration caps are a backstop, not the control (#2); enforce critical rules in deterministic hooks/code, not the prompt (#3); never escalate on self-reported confidence (#4) or sentiment (#5); surface real diagnostic errors, never generic messages (#6) or silent empty results (#7); curate ~4–7 tools per agent and split via subagents/MCP boundaries, not 18 (#8); review with a fresh-context/separate agent, never same-session self-review (#9); report per-category/per-slice metrics, never a single masking aggregate (#10). These ten map across D1 and D5 and anchor the verification and escalation questions in S1 and S5.
