# Scenario 2 — Code Generation with Claude Code (CLAUDE.md, Plan Mode, Slash Commands)

This is the deep-dive reference for **S2: Code Generation with Claude Code**. It anchors questions in Domain **D3 Claude Code Configuration & Workflows (20%)** and overlaps heavily with **D4 Prompt Engineering & Structured Output** and **D5 Context Management & Reliability**. In S2 the system under test is the Claude Code agent operating in a real repository: configuring project context, gating risky actions, encoding repeatable workflows, and verifying changes with tests. Official docs: Claude Code overview at https://docs.anthropic.com/en/docs/claude-code and best practices at https://www.anthropic.com/engineering/claude-code-best-practices.

The recurring S2 exam frame: "An architect wants Claude Code to make a change reliably and safely. Which mechanism enforces/structures/verifies it?" The right answer is almost always a **deterministic mechanism** (CLAUDE.md context, plan mode, permissions, hooks, tests) rather than a polite request buried in a prompt.

## CLAUDE.md: Structuring Project Context for Claude Code

`CLAUDE.md` is the project memory file Claude Code automatically loads into context at the start of a session. It is the single highest-leverage configuration in S2 because it is read on *every* turn — it is the durable, deterministic place to put facts the agent must not forget. Reference: https://docs.anthropic.com/en/docs/claude-code/memory.

**What belongs in CLAUDE.md** (the high-value, exam-correct contents):
- **Build / test / lint commands** — the exact commands (`npm test`, `pytest -q`, `make build`) so the agent runs the real verification step instead of guessing.
- **Code conventions** — naming, formatting rules, preferred libraries, "use X not Y" (e.g. "use the repo's `httpClient` wrapper, never raw fetch").
- **Architecture orientation** — where key modules live, the directory map, module boundaries, how layers talk to each other.
- **Workflow rules** — branch naming, commit message format, "run tests before committing," "never edit generated files in `/dist`."
- **Gotchas** — non-obvious constraints that have burned the team (a flaky migration step, a required env setup).

**What must NOT go in CLAUDE.md** (classic wrong answers):
- **Secrets / API keys / tokens / passwords.** CLAUDE.md is committed to the repo and shared; secrets belong in environment variables or a secrets manager, never in project memory. This is the single most-tested S2 trap.
- **Critical enforcement rules you assume will always be obeyed.** CLAUDE.md *guides* the model; it does not *guarantee* behavior. A line like "ALWAYS run the linter before committing" is prompt-based enforcement (anti-pattern #3). If the rule is business-critical, enforce it with a **hook** or **permission**, not a sentence.
- **Bloat.** Every token in CLAUDE.md is read every turn. Keep it tight and curated; dumping the entire style guide dilutes signal and burns context budget.

### CLAUDE.md hierarchy and imports

Claude Code merges memory from multiple scopes, most-specific-wins by proximity to the work:
- **Enterprise / system policy** (managed, org-wide).
- **User scope** `~/.claude/CLAUDE.md` — personal prefs across all your projects.
- **Project scope** `./CLAUDE.md` at repo root — shared with the team, committed.
- **Subdirectory / nested** `CLAUDE.md` — loaded when Claude works in that subtree, for module-specific rules in a monorepo.
- **Local / untracked** `CLAUDE.local.md` — personal project notes not committed (deprecated in favor of imports + user memory in newer versions).

Use `@path/to/file` **imports** inside CLAUDE.md to pull in other docs (e.g. `@docs/architecture.md`) so context is composable and DRY instead of copy-pasted. You can bootstrap a CLAUDE.md with the `/init` command, then prune it.

**Exam signal:** If a question says "Claude keeps using the wrong test command" or "ignores our naming convention," the fix is to put the command/convention in **CLAUDE.md** (project scope), not to repeat it in every prompt. If it says "secrets ended up in version control," the cause is putting them in CLAUDE.md — wrong by construction.

## Plan Mode: Safe, Reviewable Changes Before Editing

Plan mode makes Claude Code **research and propose** before it **writes**. The agent explores the codebase and produces a concrete plan (files to touch, approach, risks) that you approve *before any edit or command executes*. It is the read-only, "look before you leap" gate for non-trivial work.

**When to use plan mode (right pattern):** Any **large, multi-file, ambiguous, or risky** change — refactors, new features touching several modules, migrations, anything where the wrong approach is expensive to unwind. Plan mode separates *thinking* from *doing*, so a human reviews the strategy while it's still cheap to change. It also produces better results because the model commits to an approach instead of improvising mid-edit.

**The wrong answer:** "Skip plan mode and let Claude edit directly to save time" on a large change. For a one-line typo fix that's fine; for a cross-cutting refactor it's the tested anti-pattern — you lose the cheap review checkpoint and the agent may thrash across many files before you can course-correct.

**Exam signal:** "Architect wants to review the approach to a multi-file refactor before files change" → **plan mode**. Distractors: "raise the iteration cap," "add more detail to the prompt and hope," "review the diff afterward" (review-after is strictly worse than review-the-plan-before for large changes).

## Custom Slash Commands: Encoding Repeatable Workflows

A custom slash command is a reusable prompt template stored as a Markdown file in `.claude/commands/` (project-scoped, committed and shared) or `~/.claude/commands/` (personal). Invoking `/fix-issue` runs that template. They turn a recurring multi-step workflow into one deterministic, version-controlled entry point. Reference: https://docs.anthropic.com/en/docs/claude-code/slash-commands.

Use them to standardize things the team does repeatedly: "open a PR with our template," "triage and fix a GitHub issue," "run the full review pass," "generate a migration." Commands support **arguments** (`$ARGUMENTS`, or positional `$1`, `$2`) so one template handles many inputs. They can also be scoped to invoke specific tools.

**Slash commands vs CLAUDE.md vs subagents:**
- **CLAUDE.md** = always-on *facts/context* (loaded every turn).
- **Slash command** = on-demand *workflow/procedure* (invoked when you want it).
- **Subagent** = a separate *context + role* for a focused task (delegated work).

**Exam signal:** "The team runs the same 5-step process for every bug fix and wants it standardized and shared" → **project slash command in `.claude/commands/`** (committed). If they want it personal-only, `~/.claude/commands/`. Putting a repeatable *procedure* into CLAUDE.md is a weaker answer — CLAUDE.md is for persistent context, not an on-demand workflow you trigger.

## Permissions: Gating Risky Actions Deterministically

Permissions control what Claude Code is **allowed to do** without asking. They are configured in `.claude/settings.json` (project, committed) / `.claude/settings.local.json` (personal, untracked) / user settings, with **allow**, **ask**, and **deny** rules per tool and command pattern. This is the deterministic safety boundary — enforced by the harness, not by the model's goodwill. Reference: https://docs.anthropic.com/en/docs/claude-code/settings and the IAM/permissions docs.

Right pattern: **deny** or **ask** on destructive/irreversible operations — `git push`, `rm -rf`, production deploy commands, writing outside the repo, network calls to sensitive hosts — while **allowing** safe read/build/test commands so the agent stays fast on the 95% that's harmless. This is least-privilege for an agent.

**The wrong answer:** Relying on a CLAUDE.md sentence like "do not run destructive commands" to keep the agent safe. That's prompt-based enforcement (anti-pattern #3) — a guideline, not a guarantee. A real architect gates `git push --force` and `rm -rf` with a **permission deny/ask rule**, because deterministic gating cannot be talked around.

**Exam signal:** "How do you ensure Claude can run tests freely but cannot push to production / delete files / deploy without human approval?" → **permission rules (allow safe, ask/deny risky)**, not prompt instructions, not iteration caps.

## The Core Loop: Explore → Plan → Code → Verify

The canonical Claude Code workflow from Anthropic's best-practices guide is a four-phase loop, and S2 questions test whether you can place a mechanism in the right phase:

1. **Explore** — read relevant files, understand the codebase, gather context (no edits yet). Subagents and built-in search shine here.
2. **Plan** — propose an approach (use plan mode for big changes); get human sign-off on strategy.
3. **Code** — make the edits, ideally in small reviewable increments.
4. **Verify** — **run the tests / build / type-check** to confirm the change actually works.

**Verification is running tests, not the model asserting success.** The reliable stopping condition for a coding task is "the test suite passes" (a deterministic, externally-checkable signal), not "Claude said it's done." Parsing "I'm done" or trusting a self-reported "looks correct" is anti-pattern #1/#4 — the model's natural-language claim of completion is not the control signal; the green test run is. Give Claude the real test command (via CLAUDE.md) so it closes the loop itself, and prefer making it write or run tests as the objective truth.

**Exam signal:** "How should Claude know the coding task is complete?" → **tests pass / build succeeds** (verifiable). Distractors: "Claude reports it finished," "it hit the max iteration count," "confidence score above threshold." Iteration caps are a safety backstop (anti-pattern #2), never the primary completion control.

## Subagents for Focused Tasks

Subagents are separate Claude Code agents with their **own context window, system prompt, and curated tool set**, configured in `.claude/agents/`. Delegating a focused task (e.g. "investigate why this test flakes," "audit auth code") to a subagent keeps the main thread's context clean and gives the subtask a tight, role-appropriate toolset. Reference: https://docs.anthropic.com/en/docs/claude-code/sub-agents.

Two big wins relevant to S2:
- **Context isolation** — a noisy exploration (grepping the whole repo) happens in the subagent; only the *findings* return to the main agent, protecting the primary context budget (a D5 concern).
- **Fresh-context review** — a *separate* subagent reviewing the author agent's code does **not** inherit the author's reasoning bias. Same-session self-review is anti-pattern #9; a fresh reviewer agent is the correct pattern for independent code review.

Also: keep each agent's toolset **tight (~4–7 tools), not sprawling (18)** — too many tools per agent is anti-pattern #8. If one agent needs many capabilities, split responsibilities across subagents along clear boundaries.

**Exam signal:** "How do you review generated code without bias / without polluting the main context?" → **fresh subagent**. "Agent has 18 tools and picks wrong ones" → **split into subagents with curated toolsets**.

## Hooks: Deterministic Lint / Format / Test Gates

Hooks are shell commands the harness runs **automatically at lifecycle events** — e.g. `PreToolUse`, `PostToolUse`, `Stop`, `UserPromptSubmit`. They are how you turn "please remember to format" from a prayer into a guarantee. The classic use: a `PostToolUse` hook that runs the formatter/linter (or tests) after every file edit, so style and gates are enforced by code, deterministically, regardless of whether the model "remembered." Reference: https://docs.anthropic.com/en/docs/claude-code/hooks.

This is the resolution of anti-pattern #3 (prompt-based enforcement). The exam's recurring contrast:
- "ALWAYS run prettier after editing" written in CLAUDE.md → a *guideline* the model usually follows.
- A `PostToolUse` hook running `prettier --write` on changed files → a *guarantee* the harness enforces every time.

For **business-critical** or **must-always-happen** rules (formatting consistency, blocking edits to protected paths, mandatory test runs before stop), use a **hook**. For soft preferences and context, CLAUDE.md is fine. A hook can also *block* an action (non-zero exit on `PreToolUse`) — e.g. reject any edit to a generated file — which permissions-by-prompt cannot do.

**Exam signal:** "How do you guarantee every edit is auto-formatted / linted / type-checked?" → **hook (PostToolUse)**. "How do you guarantee a step happens regardless of the model's choices?" → **hook**, not a CLAUDE.md instruction.

## Decision Tree: Which Mechanism for S2?

Walk top-down; the first match wins:

- Need a **fact/convention/command** the agent should always know? → **CLAUDE.md** (project scope; subdir for module-specific).
- Need to **review the approach** before a large/risky change touches files? → **Plan mode**.
- Need to **block / require approval** for a dangerous action (push, deploy, rm, deploy)? → **Permissions** (deny/ask in settings.json).
- Need a step to **happen automatically every time, guaranteed** (format, lint, test gate, block protected paths)? → **Hook** (PostToolUse / PreToolUse).
- Need to **standardize a repeatable multi-step workflow** and share it? → **Custom slash command** in `.claude/commands/`.
- Need a **focused subtask with isolated context** or **unbiased review**? → **Subagent** (`.claude/agents/`).
- Need to know the task is **truly done**? → **Tests pass / build succeeds** (verifiable), not self-report, not iteration cap.

## "Ask X → Answer Y" Quick Mappings

- *"Claude keeps using the wrong build/test command."* → Put the exact command in **CLAUDE.md**.
- *"Secrets showed up in the repo."* → They were in CLAUDE.md; move to **env vars / secrets manager** (never CLAUDE.md).
- *"Ensure the team's lint rules run on every change."* → **PostToolUse hook**, not a prompt line.
- *"Stop Claude from pushing to prod / deleting files without approval."* → **Permission deny/ask rule**.
- *"Review the plan for a cross-module refactor before edits."* → **Plan mode**.
- *"Skip plan mode on a big refactor to go faster."* → **Wrong** — you lose the cheap review gate.
- *"Standardize the 5-step bug-fix process across the team."* → **Project slash command** in `.claude/commands/`.
- *"Review generated code without the author's bias / without bloating context."* → **Fresh subagent**.
- *"Agent has 18 tools and chooses poorly."* → **Curate to ~4–7; split via subagents** (anti-pattern #8).
- *"How does Claude know it's finished?"* → **Tests pass** — not "I'm done," not confidence score, not max iterations.
- *"Make a business-critical rule actually enforced, not just requested."* → **Hook or permission** (deterministic), not a CLAUDE.md sentence (anti-pattern #3).

## S2 Anti-Pattern Cheat Sheet (the distractors)

- Putting **secrets in CLAUDE.md** — wrong by construction; it's committed and shared.
- **Prompt instructions instead of hooks/permissions** for enforcement — guidance ≠ guarantee (anti-pattern #3).
- **Skipping plan mode** for large/risky changes — discards the cheap pre-edit review checkpoint.
- **Trusting "I'm done"** or a self-reported confidence score as the completion signal — use the **stop_reason / passing tests** instead (anti-patterns #1, #4).
- **Iteration cap as the primary stop** — it's a safety backstop only (anti-pattern #2).
- **Same-session self-review** of generated code — biased; use a **fresh subagent** (anti-pattern #9).
- **18 tools on one agent** — curate to ~4–7; split via subagents/MCP boundaries (anti-pattern #8).
- **Bloated CLAUDE.md** — every token loads every turn; keep it tight and use `@imports`.
