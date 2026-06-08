# Scenario 1: Customer Support Resolution Agent — Architectural Deep-Dive

This is the canonical CCA-F scenario for **agentic resolution with a human-escalation boundary**. It is built on the **Claude Agent SDK** driving an autonomous loop, **MCP tools** for backend systems (orders, accounts, refunds), and **retrieval** of policy / knowledge-base content. The exam uses it to test whether you can design *deterministic, verifiable* control flow instead of leaning on the model's vibes. Most distractors in this scenario are variations of "let the model decide based on how it feels."

## Business Problem: Autonomous Resolution vs Human Escalation

A support org wants an agent that resolves common customer issues end-to-end: look up an order, check account status, explain a policy, issue a bounded refund, update a ticket. The agent must call real backend systems through tools, and it must know **when to stop trying and hand off to a human**. The architecture question is never "can Claude write a nice reply" — it is "what governs the resolve-vs-escalate decision, and is that governance auditable, deterministic, and safe under failure?"

The two failure modes the business fears: (1) the agent confidently does the wrong thing (over-refunds, leaks account data, closes a ticket it didn't actually resolve), and (2) the agent escalates everything (defeats the purpose). Good architecture makes escalation a **measured, rule-driven decision**, not a feeling.

## Correct Architecture: Agent SDK Loop + Curated MCP Tools + Retrieval

Use the **Claude Agent SDK** to run the agentic loop. The SDK pattern is: send messages → model emits `tool_use` blocks → your harness executes the tools → you return `tool_result` blocks → repeat until the model produces a final answer with no tool calls. The loop is controlled by the API's `stop_reason`, not by parsing the model's prose. See platform.claude.com/docs (Agent SDK) and docs.anthropic.com for the tool-use loop.

Expose backend systems as **MCP tools** (Model Context Protocol, modelcontextprotocol.io). Each backend — order system, account/CRM, refund/billing — is an MCP server with a *tight, curated* tool surface. Aim for roughly **4–7 tools the agent actually needs** (e.g. `get_order`, `get_account_status`, `lookup_policy`, `issue_refund`, `update_ticket`, `escalate_to_human`). Do not dump 18 raw endpoints onto one agent; that bloats the context, raises mis-selection rate, and is the classic "too many tools" anti-pattern. If you genuinely need broad surface area, split it behind **subagents or separate MCP server boundaries**.

Policy and KB content come through **retrieval**, not memorization. The agent calls a `lookup_policy` / KB-search tool and grounds its answer in the returned text. This keeps refund rules, eligibility windows, and escalation policy *current and auditable* instead of frozen in the system prompt.

**Exam signal:** If a question describes wiring backend systems to the agent, the right answer involves **MCP servers with a curated tool set** and a retrieval tool for policy — not "put all the API docs and every endpoint in the system prompt."

## Correct Escalation Logic: Route on Measured Signals

Escalation must be driven by **verifiable, measurable signals**, evaluated in deterministic code (a hook or the harness), not by the model's self-assessment. The legitimate escalation triggers:

- **Measured task complexity** — the issue requires capabilities/tools the agent doesn't have, spans systems it can't touch, or matches a known "human-only" category (legal, fraud, account closure). Detect this from the *task structure and required actions*, not from how upset the customer sounds.
- **Tool-call failures** — a required backend call fails after retry+backoff, returns an error, or returns data that fails validation. If you can't complete the action reliably, escalate.
- **Policy / risk thresholds** — the action exceeds a hard limit (refund above the allowed amount, account-level change, anything flagged high-risk). These are enforced in code, deterministically.
- **Explicit confidence from verifiable checks** — confidence that comes from *checking reality*: did `get_order` return a matching order? Did eligibility validation pass? Did the refund amount fall under the policy cap? This is real evidence, not the model rating itself.

The discipline: every escalation reason should be something you could write an assertion about and log. "Refund amount $240 exceeds $200 auto-approve cap → escalate" is auditable. "Model felt unsure → escalate" is not.

**Exam signal:** When asked *what should trigger escalation*, pick the option grounded in **task complexity, tool failures, and policy/risk thresholds**. Reject anything mentioning customer sentiment or the model's own confidence number.

## Why NOT Sentiment, and Why NOT Self-Reported Confidence

**Sentiment is not complexity.** An angry customer with a trivial, fully-automatable problem (reset a password) does not need a human; a calm customer requesting a $5,000 refund on a closed account does. Routing on sentiment escalates the wrong cases and misses the dangerous ones. Sentiment can inform *tone of the reply*, never the *resolve-vs-escalate decision*. (Anti-pattern #5.)

**Self-reported confidence is uncalibrated.** Asking the model "how confident are you (0–1)?" and escalating below a threshold means your safety boundary is set by an unverified number the model generates about itself. It can be confidently wrong. Replace it with **confidence derived from verifiable checks** — validation passing, tool results matching, amounts under policy caps. (Anti-pattern #4.)

## Error Handling: Diagnostic Errors, Retry + Backoff, Safe Fallback

Tool errors must be **diagnostic and returned to the model**, not swallowed. When `issue_refund` fails, the `tool_result` should carry the actual error (`error_code: ACCOUNT_LOCKED`, the failing field, the constraint violated) so the model can adapt — retry differently, try another path, or escalate with context. Generic "an error occurred" strips the model of the information it needs to recover. (Anti-patterns #6, #7.)

**Never silently suppress errors.** Returning an empty result or a success-shaped payload when the call actually failed makes the agent close tickets that aren't resolved and report success that didn't happen — the most dangerous bug in a support agent. Failures must surface.

Use **retry with exponential backoff** for transient/network failures (timeouts, 429/5xx) inside the harness. Distinguish *transient* (retry) from *deterministic* (don't retry a validation rejection — escalate or fix input). After retries are exhausted, the safe behavior is a **clean fallback to a human** with the full diagnostic trail attached, not a fabricated answer.

**Exam signal:** The correct error-handling answer **returns a descriptive error in the `tool_result` so the model can react**, retries transient failures with backoff, and falls back to a human on exhaustion. Wrong answers hide the error, return empty-as-success, or retry forever.

## Guardrails via Hooks: Business Rules Are Deterministic Code

Hard business rules — **refund limits, allowed actions, PII handling, who can be escalated to** — are enforced with **hooks / deterministic code that gate tool execution**, not by instructions in the prompt. A pre-tool hook inspects the proposed `issue_refund` call; if `amount > policy_cap`, it **blocks the call and forces escalation** before any money moves. The model cannot talk its way past a code check.

Prompt-based enforcement ("Never refund more than $200") is the trap: it is probabilistic and bypassable, so it must never be the sole control on a critical business rule. Put the cap in a hook; let the prompt *describe* the rule for good behavior, but let the **hook be the enforcer**. (Anti-pattern #3.) See docs.anthropic.com/en/docs/claude-code for the hooks model; the same pre-tool-gate concept applies to Agent SDK harnesses.

**Exam signal:** "How do you guarantee the agent never exceeds the refund cap?" → a **deterministic pre-tool hook / code check that blocks the call**, not a stronger system-prompt sentence.

## Termination: Loop on `stop_reason`, Not on Prose

The agent stops when the API says it's done. Continue the loop while `stop_reason == "tool_use"` (the model wants another tool). Finish when `stop_reason == "end_turn"` (final answer, no tool calls). Handle `max_tokens` and `pause_turn` explicitly. **Do not** parse the assistant's text for phrases like "I'm done" or "resolved" to decide when to stop — natural-language termination is brittle and gameable. (Anti-pattern #1.)

An **iteration cap is a safety backstop, not the primary control.** Cap total turns to prevent runaway loops, but the real stopping mechanism is `stop_reason` plus your explicit escalation/completion conditions. Treating the cap as the main loop control is anti-pattern #2.

**Exam signal:** "How does the loop know to stop?" → **check `stop_reason`**. Distractors: "when the model says it finished," "after N iterations" (as the primary mechanism).

## Decision Tree: Resolve vs Escalate

```
Customer issue arrives
        │
        ▼
Classify required actions (from task structure, not sentiment)
        │
        ├─ Needs human-only category (legal/fraud/account-closure)? ──► ESCALATE
        │
        ▼
Run agent loop (Agent SDK), call curated MCP tools
        │
        ▼
For each tool call:
   ├─ Transient failure? ──► retry + backoff
   │        └─ exhausted? ──► ESCALATE (attach diagnostics)
   ├─ Deterministic/validation failure? ──► ESCALATE (attach diagnostics)
   └─ Success ──► continue
        │
        ▼
Action exceeds policy/risk threshold (e.g. refund > cap)?
   └─ YES ──► pre-tool HOOK blocks call ──► ESCALATE
        │
        ▼
Verifiable checks pass (order matches, eligibility valid, amount under cap)?
   ├─ NO  ──► ESCALATE
   └─ YES ──► execute action, update ticket
        │
        ▼
stop_reason == end_turn?  ──► RESOLVE & close
(iteration cap = backstop only)
```

## Likely WRONG Answers (Distractors to Eliminate)

- **Escalate based on customer sentiment / tone analysis.** Sentiment ≠ complexity. (Anti-pattern #5.)
- **Escalate when the model's self-reported confidence is low.** Uncalibrated; use verifiable checks instead. (Anti-pattern #4.)
- **Enforce the refund cap by instructing the model in the system prompt.** Use a deterministic hook. (Anti-pattern #3.)
- **Return empty results / generic errors on tool failure**, or swallow errors silently. Surface diagnostic errors so the model can react and fall back safely. (Anti-patterns #6, #7.)
- **Stop the loop when the model says "I'm done."** Use `stop_reason`. (Anti-pattern #1.)
- **Use an iteration cap as the main termination control.** It's a backstop. (Anti-pattern #2.)
- **Give the one agent 18 backend tools.** Curate to ~4–7; split via subagents / MCP boundaries. (Anti-pattern #8.)
- **Put all policy text in the system prompt.** Retrieve it via a tool so it stays current and auditable.

## "If the question asks X, the answer is Y" Mappings

- **If the question asks** "what should drive the resolve-vs-escalate decision?" → **measured task complexity, tool-call failures, and policy/risk thresholds** — never sentiment, never self-reported confidence.
- **If the question asks** "how do you guarantee the refund cap is never exceeded?" → a **deterministic pre-tool hook / code check that blocks the call**, not a prompt instruction.
- **If the question asks** "a backend tool call fails — what should the agent do?" → **return a descriptive diagnostic error in the `tool_result`, retry transient failures with backoff, and fall back to a human on exhaustion** — not hide it or return empty success.
- **If the question asks** "how does the agent loop know when to stop?" → **inspect the API `stop_reason`** (continue on `tool_use`, finish on `end_turn`); the iteration cap is only a safety backstop.
- **If the question asks** "how should backend systems be exposed to the agent?" → **MCP servers with a tight, curated tool set (~4–7) plus a retrieval tool for policy/KB** — not 18 raw endpoints or policy baked into the prompt.
- **If the question asks** "the customer is furious about a trivial password reset — escalate?" → **no; tone is not complexity.** Resolve it; sentiment only shapes reply tone.
