# Domain 3 — Claude Code Configuration & Workflows (20%)

This domain tests whether you can configure Claude Code correctly for a team and drive a disciplined, AI-first development workflow. It is anchored mostly to **S2 (Code Generation with Claude Code)** and **S5 (Claude Code for CI/CD)**. The exam rewards answers that push enforcement into **deterministic mechanisms** (settings, permissions, hooks, headless flags) instead of "just tell the model to" prose. Source documentation: Claude Code docs at https://docs.anthropic.com/en/docs/claude-code and the settings/memory/hooks/slash-command pages within it.

The recurring throughline of D3: **memory shapes behavior, settings constrain behavior, hooks enforce behavior, and headless mode automates behavior.** Knowing which layer a requirement belongs to is most of the exam.

## CLAUDE.md Memory Hierarchy and Precedence

Claude Code loads memory from multiple `CLAUDE.md` files and concatenates them into the system context. The hierarchy, from highest authority to lowest, is: **enterprise/managed policy → project memory → user memory → local project memory.** Higher-precedence files are not "overwritten" so much as they take priority when guidance conflicts, and enterprise/managed policy is the only layer an end user cannot edit.

- **Enterprise / managed policy** lives in a system-level path managed by an organization (deployed via MDM/IT), applies to every user on the machine, and cannot be overridden by a developer. Put org-wide security and compliance mandates here.
- **Project memory** is `./CLAUDE.md` at the repo root (committed to git). It is shared with the whole team and is the right home for architecture conventions, build/test commands, code style, and "how we work in this repo."
- **User memory** is `~/.claude/CLAUDE.md` — personal, cross-project preferences for a single developer (e.g., "always run the linter," preferred commit style). It is not shared with teammates.
- **Local project memory** is the lowest-precedence, gitignored layer for personal, repo-specific notes that you do not want to commit (historically `CLAUDE.local.md`; the modern, preferred mechanism is a gitignored import rather than a separate filename).

Claude Code also discovers `CLAUDE.md` files in **subdirectories** as it reads files in those trees, so monorepos can scope memory per package. Memory is loaded top-down from the root toward cwd plus any relevant subtree files.

**Exam signal:** A scenario says "all repos in the company must never call the production database from generated code." The correct placement is **enterprise/managed policy** (developers cannot remove it), not project `CLAUDE.md` (a developer could edit it) and definitely not a per-developer user memory. Watch for distractors that put a non-negotiable security rule in a layer a user can edit — or that rely on memory at all when the real answer is a **hook** (see below).

### What Belongs in CLAUDE.md vs. Elsewhere

CLAUDE.md is for **stable, high-signal context the model should always have**: build/test/lint commands, repo structure, naming conventions, "do/don't" coding rules, and pointers to key files. Keep it concise — bloated memory wastes context and dilutes attention. It is **guidance**, not enforcement: the model usually follows it but is not guaranteed to. Anything that *must* hold (security gates, "never commit secrets," "block edits to `/infra`") belongs in **hooks or permissions**, because those are deterministic and code-enforced rather than probabilistic.

**Exam signal:** "The team keeps reminding Claude in CLAUDE.md not to edit migration files, but it still does occasionally." The right fix is a **PreToolUse hook** (or a deny permission rule) that blocks edits to that path — not stronger wording in the prompt. This is anti-pattern #3 (prompt-based enforcement of critical rules).

## @-Imports in CLAUDE.md

CLAUDE.md supports **`@path/to/file` imports** so memory can be modularized and composed instead of duplicated. An import line pulls another file's content into the loaded memory. This is how you keep a lean root `CLAUDE.md` that imports shared standards (`@docs/coding-standards.md`), and how you wire up **local, gitignored personal memory**: commit a `CLAUDE.md` that `@`-imports a gitignored file like `@CLAUDE.local.md` so each developer's private notes load without being committed. Imports can be nested to a bounded depth, and `@` references inside code spans/code blocks are not treated as imports.

**Exam signal:** A question asks how to share one canonical style guide across many repos or keep memory DRY. The answer is **@-imports**, not copy-pasting the same block into every `CLAUDE.md`.

## settings.json and the Permission Model

Claude Code is configured by `settings.json` files that cascade by scope. From highest to lowest precedence: **enterprise/managed settings → command-line arguments → project `.claude/settings.local.json` (gitignored, personal) → project `.claude/settings.json` (committed) → user `~/.claude/settings.json`.** Enterprise settings always win (an org locks down what developers can change), and note the high-yield nuance: the gitignored **Local** file *outranks* the committed **Project** file, so a developer can locally override shared project settings. Permission rule arrays additionally *merge* across scopes rather than fully overriding, with `deny` > `ask` > `allow` evaluated on the merged set. `settings.json` controls permissions, hooks, environment variables, model selection, and MCP server enablement.

The **permission model** governs every tool call. Rules are expressed as `allow`, `ask`, and `deny` lists keyed by tool and (for Bash/edits) by pattern — e.g., `Bash(npm run test:*)` to allow a command family, or `Edit(./src/**)` to scope file edits. Resolution prioritizes **deny over ask over allow**: a `deny` rule always blocks, even if an `allow` rule would otherwise match. This is what makes `deny` the correct tool for hard prohibitions like `Read(./.env)` or blocking edits to protected paths.

- **allow** — auto-approve matching tool calls (no prompt). Use for safe, high-frequency operations (running tests, reading source).
- **ask** — prompt the user for confirmation each time (the interactive default for anything not explicitly allowed/denied).
- **deny** — hard-block the call regardless of mode or other rules. Use for secrets, destructive commands, and protected paths.

**Exam signal:** "Developers should be able to run the test suite without confirmation but must never read `.env` or run `rm -rf`." Correct answer combines an **allow** rule for the test command pattern and **deny** rules for the secret file and destructive command — and notes that deny takes precedence. A distractor will suggest only writing instructions in CLAUDE.md.

### Permission Modes: default, acceptEdits, plan, bypassPermissions

Permission **modes** set the baseline behavior for a session, on top of the rule lists:

- **default** — normal interactive prompting; unmatched actions trigger an `ask`.
- **acceptEdits** — auto-accepts file edits (and similar low-risk filesystem writes) so you stop confirming every change, while still gating riskier actions. Good for trusted, fast iteration.
- **plan** — read-only exploration mode: Claude can read, search, and analyze but **cannot edit files or run mutating commands**; it produces a plan for approval before any changes. Ideal for safe investigation of unfamiliar code.
- **bypassPermissions** — skips prompts entirely (dangerous; appropriate only in sandboxed/CI contexts you fully control).

**Exam signal:** A scenario wants Claude to "analyze the codebase and propose a refactor approach before touching anything." That is **plan mode**. If the scenario then wants rapid, low-friction application of an already-approved plan, that is **acceptEdits**. Do not confuse plan mode (no writes, produce a plan) with acceptEdits (writes auto-approved).

## Custom Slash Commands vs. Built-ins

Built-in slash commands ship with Claude Code (e.g., `/init`, `/clear`, `/review`, `/help`, `/agents`, `/mcp`, `/permissions`) and control the session itself. **Custom slash commands** are user-defined, reusable prompts stored as **Markdown files** in `.claude/commands/` (project-scoped, committed/shareable) or `~/.claude/commands/` (user-scoped, personal). The filename becomes the command name: `.claude/commands/fix-issue.md` → `/fix-issue`. Subdirectories namespace commands.

A custom command file can include **YAML frontmatter** to declare metadata such as `description`, `allowed-tools` (restrict which tools the command may use), `argument-hint`, and (where supported) a `model`. The body is the prompt. **`$ARGUMENTS`** interpolates everything the user typed after the command; positional **`$1`, `$2`, …** capture individual args. Commands can also run shell with `!`-prefixed lines and reference files with `@`, so a command can inject live context (e.g., `!git diff`) into its prompt.

**Exam signal:** "The team repeats the same multi-step 'create a PR with changelog and run tests' prompt constantly." The right answer is a **custom slash command** (Markdown in `.claude/commands/`, committed so the whole team gets it) using `$ARGUMENTS`/frontmatter — not retyping the prompt and not stuffing it in CLAUDE.md. Know that custom commands live in `.claude/commands/`, while **subagents** live in `.claude/agents/` (don't swap them).

## HOOKS — Deterministic Enforcement of Business and Security Rules

Hooks are **shell commands Claude Code runs automatically at defined lifecycle events**, configured in `settings.json`. Because they are deterministic code that executes outside the model's discretion, they are the exam's canonical answer for *enforcing* anything that must always happen — the antidote to anti-pattern #3 (prompt-pleading). Key events include:

- **PreToolUse** — runs **before** a tool executes and can **block** it. A hook that exits with **code 2** blocks the tool call and feeds stderr back to Claude as feedback; other non-zero codes surface a non-blocking error. This is how you enforce "no edits to `/infra`," "no `git push --force`," or "scan the command for secrets before it runs."
- **PostToolUse** — runs **after** a tool succeeds. Use it for auto-formatting, running the linter/tests on a changed file, or logging. It can also return feedback to Claude.
- **UserPromptSubmit** — runs when the user submits a prompt (can inject context or block).
- **SessionStart / SessionEnd**, **Stop / SubagentStop**, **PreCompact**, **Notification** — lifecycle points for setup, cleanup, and guardrails (e.g., Stop hooks can prevent ending while work is incomplete).

Hook decisions can be driven by **exit codes** (2 = block) or by emitting **structured JSON** on stdout (e.g., `{"decision": "block", "reason": "..."}` / permission decisions) for finer control. Hooks receive event data (tool name, inputs, file paths) on stdin so they can make path- or command-specific decisions.

**Why hooks beat prompt-pleading:** A prompt instruction ("please always run the formatter") is **probabilistic** — the model may forget, especially under long context or distraction. A PostToolUse formatting hook or a PreToolUse blocking hook is **guaranteed** to fire. For *critical business rules and security gates*, deterministic enforcement is mandatory; "tell the model in CLAUDE.md" is the wrong answer.

**Exam signal:** "Generated code must be auto-formatted and must never write to the secrets directory." Correct design = **PostToolUse hook** that runs the formatter on edited files **plus** a **PreToolUse hook (exit code 2)** or `deny` permission blocking writes to that directory. If an answer choice says "add a strong instruction to CLAUDE.md telling Claude to always format and avoid secrets," that is the trap (anti-pattern #3).

**Exam signal:** A question contrasts where to put a rule. Heuristic to memorize — **Guidance → CLAUDE.md; Permission boundary → settings allow/ask/deny; Guaranteed action/veto → hook.** The harder the requirement (security, compliance, "must never/always"), the lower-level and more deterministic the mechanism must be.

## Subagents Configuration

Subagents are specialized agents Claude Code can delegate to, defined as **Markdown files with YAML frontmatter in `.claude/agents/`** (project) or `~/.claude/agents/` (user). Frontmatter typically includes `name`, `description` (which drives when Claude auto-delegates to it), an optional restricted **`tools`** list, and sometimes a `model`. The body is the subagent's system prompt. Each subagent runs in its **own isolated context window**, which both protects the main thread from context pollution and enables **independent, bias-free review** (directly addresses anti-pattern #9: same-session self-review inherits the author's reasoning bias). You manage them via the `/agents` command.

**Exam signal:** "Limit a code-review agent to read-only tools and give it a fresh context separate from the implementer." That is a **subagent** with a restricted `tools` list in its frontmatter — and the fresh context is *why* it's a better reviewer than asking the same session to grade its own work.

## MCP Servers in Claude Code

Claude Code is an MCP client and connects to **MCP servers** to gain external tools, resources, and prompts. Servers are configured at **local (user), project, or enterprise** scope; **project-scoped servers are declared in a committed `.mcp.json`** at the repo root so the whole team shares the same toolset. Transports include **stdio** (local subprocess), **SSE**, and **HTTP** (remote). You add/inspect servers with `claude mcp add ...` and the **`/mcp`** command (status, auth/OAuth, listing tools). Permissions still apply: MCP tool calls go through the same allow/ask/deny model, and enterprise settings can restrict which servers are allowed.

**Exam signal:** "Everyone on the team should automatically have the Jira and Postgres MCP tools when they open this repo." The answer is a committed **`.mcp.json`** (project scope), not each developer adding servers to their personal user config. Tie this to S4 (Developer Productivity) and S1 (support agent tools) where MCP is the integration boundary.

## Headless Mode (`claude -p`) for CI/CD and Scripting

**Headless / non-interactive mode** runs Claude Code as a one-shot, scriptable process via **`claude -p "<prompt>"`** (print mode). It is the foundation of S5 (CI/CD): no TUI, takes a prompt, runs to completion, prints output, exits. Key flags and behaviors to know:

- **`-p` / `--print`** — non-interactive; print the result and exit (pipeable: `cat diff | claude -p "review this"`).
- **`--output-format`** — `text`, `json`, or `stream-json` for **machine-readable** output you can parse in a pipeline (essential for gating a build on structured results — ties to D4 structured output).
- **`--allowedTools` / `--permission-mode`** — pre-authorize tools so the run doesn't block on prompts in CI; for fully unattended jobs you constrain tools tightly rather than disabling all safety.
- Useful for **scheduled/automated** tasks: triaging issues, generating release notes, multi-pass code review in a pipeline.

**Exam signal:** "Run an automated code review on every PR and fail the build if critical issues are found." Correct answer = **headless `claude -p`** with **`--output-format json`** so the CI job can parse the verdict and set an exit status — not an interactive session and not parsing free-form prose. Pre-authorize only the needed tools. Note: headless mode does **not** persist conversation between invocations unless you explicitly resume/continue a session (`--continue`/`--resume`); each CI invocation is fresh by default.

## Plan Mode for Safe Exploration

Plan mode (enter with **Shift+Tab** to cycle modes, or start with `--permission-mode plan`) makes Claude **read-only**: it investigates, reads, searches, and proposes a concrete plan, but performs **no edits and no mutating commands** until you approve. It is the safe default for unfamiliar codebases, risky refactors, and "understand before you change" tasks. After approval you switch to a writing mode (default or acceptEdits) to execute.

**Exam signal:** Any scenario emphasizing "explore safely first," "don't change anything until I approve the approach," or "investigate this legacy module" points to **plan mode**. Distractors offer acceptEdits (wrong — that auto-writes) or "just trust the model."

## AI-First Dev Workflows: Explore → Plan → Code → Verify

The recommended Claude Code workflow is a disciplined loop: **Explore** (read relevant files/use plan mode to build context — resist editing first), **Plan** (have Claude produce an approach; use extended thinking for hard problems), **Code** (implement against the approved plan), **Verify** (run tests, linters, and an independent review). Verification is the part the exam stresses: rely on **deterministic checks** (tests, type-checks, hooks) and a **fresh-context reviewer** rather than the implementing session's self-assessment.

**Multi-pass review (S5):** Quality comes from separate passes with separate concerns — e.g., a correctness pass, a security pass, a style/lint pass — and ideally a **fresh context or separate subagent** for the review pass so it does not inherit the author's assumptions (anti-pattern #9). In CI, each pass can be a headless `claude -p` invocation emitting structured output that gates the build, and batch processing can fan review across many files. Aggregate "looks good" is not enough; evaluate per concern and per file so a passing average doesn't hide a critical failure in one slice (anti-pattern #10).

**Exam signal:** "Improve review quality for AI-generated PRs." Best answer = **multi-pass review with a separate/fresh-context reviewer** (subagent or new headless run), each pass deterministic and structured — not "ask the same chat to double-check its own work."

## Top Traps for Domain 3

- **Prompt-pleading instead of hooks/permissions** for critical rules (anti-pattern #3). Hard rules → PreToolUse hook (exit code 2) or `deny`. CLAUDE.md is guidance, not a guarantee.
- **Putting a non-negotiable security rule in a user-editable layer.** Org mandates belong in **enterprise/managed** memory and settings, which developers cannot override; project `CLAUDE.md` can be edited away.
- **Confusing plan mode with acceptEdits.** Plan = read-only, produce a plan; acceptEdits = auto-apply writes. They are opposites in risk posture.
- **Self-review in the same session** (anti-pattern #9). Use a subagent / fresh context with restricted, read-only tools.
- **Parsing free-form prose in CI** instead of `--output-format json` from headless mode, then keying the build's exit status on the structured verdict.
- **Per-developer MCP/setup for team needs.** Shared tooling → committed `.mcp.json` and project `.claude/settings.json`; shared commands → `.claude/commands/`.
- **Mixing up directories:** slash commands in `.claude/commands/`, subagents in `.claude/agents/`, MCP servers in `.mcp.json`, behavior/permissions in `.claude/settings.json`, memory in `CLAUDE.md`.
- **Forgetting deny-precedence.** `deny` beats `ask` beats `allow`; a broad allow does not unblock a denied secret path.
- **Treating iteration caps or "I'm done" text as control flow** even inside Claude Code automation (anti-patterns #1 and #2) — stops should be driven by deterministic signals (tests passing, stop_reason, hook decisions), not vibes.
