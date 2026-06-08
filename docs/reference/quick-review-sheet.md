# CCA-F Quick Review Sheet

Use this file for last-mile review after you have read the domain and scenario guides.

## Format

- 60 scenario-based multiple-choice questions.
- 120 minutes.
- Scaled score from 100-1000.
- Passing score: 720.
- No penalty for guessing.
- 6 scenario families exist; 4 appear in a sitting.

## Domain Weights

| Domain | Topic | Weight |
| --- | --- | --- |
| D1 | Agentic Architecture & Orchestration | 27% |
| D2 | Tool Design & MCP Integration | 18% |
| D3 | Claude Code Configuration & Workflows | 20% |
| D4 | Prompt Engineering & Structured Output | 20% |
| D5 | Context Management & Reliability | 15% |

## Highest-Yield Rules

- Start with the simplest architecture that works.
- Workflow = predefined code path.
- Agent = model dynamically directs process and tool use.
- Multi-agent only when parallel breadth or context isolation is worth the cost.
- Agent loops branch on `stop_reason`, not natural language.
- `tool_use` means run the tool and return `tool_result`.
- `end_turn` means the model ended its turn normally.
- `max_tokens` means truncation; handle it, do not treat it as success.
- Critical business rules belong in deterministic code or hooks.
- Escalation should depend on deterministic conditions, not sentiment or self-reported confidence.
- Tool sets should be small and distinct, roughly 4-7 tools per agent context.
- Tool errors should return actionable diagnostic context.
- MCP is an integration boundary for exposing tools, resources, and prompts.
- `CLAUDE.md` carries project memory; hooks enforce; plan mode asks before edits.
- Structured output reliability comes from schema/tool enforcement plus validation and retry.
- Prompt caching works best with stable content before volatile content.
- Batch is for high-volume, latency-tolerant work.
- Evals must report per-slice quality, not only aggregate accuracy.

## Classic Wrong-Answer Patterns

1. Parsing "I'm done" to stop an agent loop.
2. Using an iteration cap as the primary completion signal.
3. Enforcing critical rules only through prompt text.
4. Escalating based on model confidence.
5. Escalating based only on customer sentiment.
6. Returning vague tool errors.
7. Hiding failures as empty success.
8. Giving one agent too many tools.
9. Reviewing in the same context that produced the work.
10. Reporting only aggregate metrics.

## How To Attack A Question

1. Identify the domain: D1 loop/architecture, D2 tools/MCP, D3 Claude Code, D4 prompting/output, or D5 context/reliability.
2. Identify the failure mode.
3. Eliminate options that use soft/natural-language control where deterministic control is available.
4. Prefer the smallest design that satisfies the stated scenario constraints.
5. If torn, choose the option that fails loudly, validates explicitly, and preserves observability.

## Scenario Cues

- S1 support agent: escalation, identity/refund prerequisites, MCP tools, human approval.
- S2 code generation: `CLAUDE.md`, plan mode, slash commands, deterministic project rules.
- S3 multi-agent research: coordinator/subagent split, context isolation, aggregation, error surfacing.
- S4 developer productivity: codebase exploration, curated tool sets, MCP boundaries.
- S5 CI/CD: headless Claude Code, structured outputs, batch, fresh-context review.
- S6 structured extraction: schemas, validation-retry, per-document-type evals.

## Final Drill

- Run mixed scenario questions at 2 minutes each.
- For each miss, write down the domain, anti-pattern, and deterministic control you overlooked.
- Re-read the corresponding domain guide before doing more questions.

