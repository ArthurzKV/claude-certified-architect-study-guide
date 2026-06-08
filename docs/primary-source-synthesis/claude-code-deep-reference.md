# Claude Code Deep Reference (CCA-F: Domain D3, Scenarios S2 & S5)

This is an original study reference for the Claude Certified Architect – Foundations exam. It covers Claude Code configuration and workflow internals as documented at docs.anthropic.com/en/docs/claude-code (now served from code.claude.com/docs). Domain D3 (Claude Code Configuration & Workflows) is 20% of the exam and is anchored mainly to scenario S2 (Code Generation with Claude Code) and S5 (Claude Code for CI/CD). Memorize the precedence rules, the deterministic-enforcement boundary (hooks/settings vs CLAUDE.md), and headless mode — these are the heavily tested seams.

## The CLAUDE.md Memory Hierarchy and Load Order

Claude Code starts every session with a fresh context window. CLAUDE.md files are the mechanism that carries persistent project/user/org instructions into that fresh window. They are loaded at the start of every session as a user message after the system prompt — not as part of the system prompt — which is precisely why they shape behavior but do not hard-enforce it.

The hierarchy, from broadest scope to most specific, is: (1) **Managed policy** — `/Library/Application Support/ClaudeCode/CLAUDE.md` (macOS), `/etc/claude-code/CLAUDE.md` (Linux/WSL), `C:\Program Files\ClaudeCode\CLAUDE.md` (Windows); (2) **User instructions** — `~/.claude/CLAUDE.md`; (3) **Project instructions** — `./CLAUDE.md` or `./.claude/CLAUDE.md`; (4) **Local instructions** — `./CLAUDE.local.md` (gitignored, personal per-project). Managed policy loads first and cannot be excluded by individual settings.

Critically, CLAUDE.md files are **concatenated, not overridden**. Claude walks up the directory tree from the working directory and loads every CLAUDE.md and CLAUDE.local.md it finds. Content is ordered root-down: a parent `foo/CLAUDE.md` appears in context *before* `foo/bar/CLAUDE.md`, so instructions closest to where you launched are read last. Within a directory, `CLAUDE.local.md` is appended after `CLAUDE.md`. Subdirectory CLAUDE.md files load on demand when Claude reads files in those subdirectories, not at launch.

Managed CLAUDE.md content can also be embedded directly in `managed-settings.json` via the `claudeMd` key — honored only in managed/policy settings, ignored in user/project/local. In monorepos, `claudeMdExcludes` (glob patterns against absolute paths) skips irrelevant ancestor files, but managed-policy CLAUDE.md can never be excluded.

**Exam signal (S2):** When a scenario asks "where do team-shared coding standards belong vs. a personal sandbox URL," the answer is project `./CLAUDE.md` (committed) for the team standard and `CLAUDE.local.md` (gitignored) for personal data. When it asks about an organization-wide security policy that must apply on every machine, the answer is the managed-policy location, because it cannot be overridden.

## Importing Files with @path Syntax

A CLAUDE.md can pull in additional files with `@path/to/import` syntax — for example `See @README for overview and @package.json for npm commands`, or `- @~/.claude/my-project-instructions.md`. Imported files are expanded and loaded into context at launch alongside the referencing file. Both relative paths (resolved relative to the importing file, not the cwd) and absolute/`~` paths are allowed. Imports recurse up to a maximum depth of four hops.

A key exam nuance: **imports help organization but do not save context** — every imported file is loaded in full at launch. So splitting a giant CLAUDE.md into `@`-imports does not reduce token consumption. To actually load instructions conditionally, use path-scoped `.claude/rules/*.md` files with `paths:` frontmatter, which only enter context when Claude touches matching files. The first time a project uses external imports, an approval dialog appears; declining disables them permanently for that project.

## settings.json Hierarchy and Precedence

Settings control behavior deterministically (unlike CLAUDE.md, which only guides). The precedence order, highest to lowest, is: (1) **Managed** policy settings (`managed-settings.json`, e.g. `/Library/Application Support/ClaudeCode/managed-settings.json` on macOS) — cannot be overridden; (2) **Command-line arguments**; (3) **Local** project settings (`.claude/settings.local.json`, gitignored); (4) **Project** settings (`.claude/settings.json`, committed); (5) **User** settings (`~/.claude/settings.json`).

For scalar settings, the highest-priority scope wins outright. The **critical exception**: permission rules (`allow`/`ask`/`deny`) **merge across all scopes** rather than overriding. Every allow, ask, and deny from every applicable scope combines. This is a favorite distractor — a user-level `allow` does not get "overwritten" by a project file; both rules coexist.

**Exam signal:** If a question says an organization must guarantee `Bash(curl *)` is blocked on every developer machine regardless of personal config, the answer is a `permissions.deny` rule in managed settings — enforced by the client, not negotiable.

## Permissions: allow / ask / deny and Evaluation Order

Permission rules use the form `Tool` or `Tool(specifier)`, e.g. `Bash(npm run test *)`, `Read(./.env)`, `WebFetch(domain:example.com)`, `Edit(./src/**)`. Evaluation is **first-match-wins in a fixed order: deny first, then ask, then allow.** Deny is the most restrictive and always pre-empts an allow. So a `deny` on `Read(./.env)` beats any broad `allow`, which is the secure default for secrets.

### Permission Modes (incl. Plan Mode and acceptEdits)

Modes are set via `--permission-mode` or the `defaultMode` setting. **default** prompts on each action matching `ask` rules. **acceptEdits** auto-accepts file edits without prompting (other restrictions still apply) — used to speed up trusted refactors. **plan** (plan mode) has Claude research and produce a plan for review *before any mutation*; it does not edit files or run mutating commands until you approve, making it the safe exploration mode. **bypassPermissions** skips all checks (dangerous; can be locked off with `disableBypassPermissionsMode: "disable"`).

**Exam signal (S2):** Plan mode is the right answer whenever a scenario wants Claude to explore a codebase and propose changes for human review before touching anything — e.g. a risky migration. acceptEdits is the answer for an already-scoped, low-risk batch of edits where prompting on each one is friction. Bypass is almost never the correct answer in an exam (it's the unsafe distractor).

## Slash Commands: Built-in and Custom

Built-in commands include `/init` (analyze the codebase and generate or improve a starting CLAUDE.md — it suggests improvements rather than overwriting an existing one, and incorporates an existing AGENTS.md/.cursorrules), `/clear` (wipe the conversation/context for a fresh start), `/compact` (summarize and compress the conversation to reclaim context while preserving intent), `/memory` (list and edit loaded CLAUDE.md / CLAUDE.local.md / rules files), `/model`, and `/config`.

**Custom slash commands** are Markdown files placed in `.claude/commands/` (project, committed) or `~/.claude/commands/` (user). The filename becomes the command name. They support YAML frontmatter (e.g. `description`, `argument-hint`, `allowed-tools`, `model`) and argument substitution: `$ARGUMENTS` for the whole argument string and `$1`, `$2`, … for positional args. They can also run shell with `!` prefixes and reference files with `@`. This is the standard way to package a repeatable, parameterized workflow (e.g. a `/fix-issue` command that takes a ticket number).

**Exam signal (S2/S5):** A reusable, parameterized review or fix routine = a custom Markdown command in `.claude/commands/` with frontmatter and `$ARGUMENTS`. Do not confuse a slash command (a prompt template) with a hook (deterministic shell enforcement) or a subagent (isolated-context delegate).

## Subagents (.claude/agents)

Subagents are defined as Markdown-with-frontmatter files in `.claude/agents/` (project) or `~/.claude/agents/` (user). Each runs in its own **isolated context window** with its own system prompt, and can be restricted to a curated **tool subset** via frontmatter. Frontmatter typically includes `name`, `description` (used by the coordinator to decide when to delegate), and `tools` (the allowlist).

Subagents serve two exam-relevant purposes. First, **context isolation**: a noisy investigation (grep sweeps, log trawls) runs in the subagent and only the distilled findings return to the main thread, keeping the parent context clean. Second, **tool-budget management and fresh-context review**: rather than overload one agent with 18 tools, split responsibilities across subagents with tight tool sets, and run a *separate* reviewer subagent so the review does not inherit the author's reasoning bias.

**Exam signal (S3/S5):** Subagents are the canonical fix for two anti-patterns: "too many tools on one agent" (split via subagents/MCP boundaries to ~4–7 tools each) and "same-session self-review" (use a fresh-context reviewer agent). Tool restriction lives in agent frontmatter, not in the prompt body.

## Hooks: Deterministic Lifecycle Enforcement

Hooks are shell commands (or `http`, `mcp_tool`, `prompt`, `agent` handlers) that fire at fixed lifecycle events. They are configured in settings (`~/.claude/settings.json`, `.claude/settings.json`, `.claude/settings.local.json`, managed policy, or plugin `hooks/hooks.json`). The structure nests event → matcher group → handler list, where `matcher` selects tools (exact like `Bash`, list like `Edit|Write`, regex, or `*`/omitted for all).

Key events: **PreToolUse** (before a tool runs — can block it), **PostToolUse** (after a tool succeeds), **UserPromptSubmit** (before Claude processes a submitted prompt), **Stop** (when Claude finishes responding), **SubagentStop**, **SessionStart**, **PreCompact/PostCompact**, **Notification**, and **InstructionsLoaded** (logs which instruction files loaded — useful for debugging). The exam typically tests the first four.

### Hook Control Flow: Exit Codes and JSON

Exit-code control: **exit 0** = success (stdout parsed for JSON control fields). **Exit 2 = blocking error** — for blockable events (PreToolUse, UserPromptSubmit, Stop, SubagentStop, PreCompact, etc.) this stops the action, and stderr is fed back to Claude as feedback. Note that **exit code 1 is a non-blocking error**, not a block — only exit 2 blocks. For richer control, return JSON on stdout: universal fields include `continue` (false stops Claude entirely), `stopReason`, `suppressOutput`, and `systemMessage`. For PreToolUse specifically, `hookSpecificOutput.permissionDecision` can be `allow | deny | ask | defer` with a `permissionDecisionReason`. A `decision: "block"` with a `reason` is used by UserPromptSubmit/PostToolUse/Stop.

**Hooks are the mechanism for DETERMINISTIC enforcement.** The docs are explicit: CLAUDE.md and memory are context, not enforced configuration — "to block an action regardless of what Claude decides, use a PreToolUse hook." This is the single most important D3 concept.

**Exam signal (D3 / anti-pattern #3):** Whenever a scenario says a *critical business rule* must always hold — never push to main, always run lint/tests before commit, block `rm -rf`, redact secrets — the correct answer is a **hook (or a permissions deny rule)**, never "add a stronger instruction to CLAUDE.md / plead in the prompt." Prompt-based enforcement for critical rules is a classic wrong answer. Use a PreToolUse hook returning exit 2 (or `permissionDecision: deny`) to block; use PostToolUse to run a linter/formatter after edits; use Stop to gate completion on a passing check.

## MCP Integration in Claude Code

Claude Code connects to Model Context Protocol servers to gain external tools/resources. Project-scoped servers are declared in **`.mcp.json`** at the repo root (committed, team-shared). MCP scopes are **local** (personal, this project only), **project** (`.mcp.json`, shared via source control), and **user** (across all your projects). Servers can be stdio, SSE, or HTTP. Use MCP boundaries to keep each agent's tool set tight and curated rather than piling every tool onto one agent.

**Exam signal (D2/S4):** Team-shared MCP server config that should be in version control = `.mcp.json` (project scope). Personal experimental servers = local scope.

## Headless / Non-Interactive Mode (claude -p)

`claude -p "<prompt>"` (the `--print` flag) runs Claude Code non-interactively: it executes the prompt and prints the result without an interactive session — the foundation for scripting and CI/CD. Combine with `--output-format json` (or `stream-json`) for machine-parseable output, `--allowedTools`/`--permission-mode` to scope what it may do unattended, and `--append-system-prompt` for per-invocation system instructions. This is how Claude Code is embedded in pipelines, pre-commit checks, and batch review jobs.

**Exam signal (S5 / CI-CD):** Any "run Claude Code in a GitHub Action / pipeline" scenario points to headless `claude -p` with a structured output format and an explicit permission scope (allowed tools + non-interactive mode), plus deterministic gating via hooks/exit codes. Pair multi-pass review (a second fresh-context reviewer pass) with structured output so the pipeline can parse pass/fail. For large fan-out grading or extraction across many inputs, the Batch API (on the Claude API side) is the throughput tool — distinct from Claude Code's own `claude -p` orchestration.

## Output Styles, Checkpoints/Rewind, and Background Tasks

**Output styles** change how Claude Code communicates (e.g. a terse/explanatory tone) without changing its tools; `outputStyle` is one of the few settings that needs a `/clear` or restart to rebuild. **Checkpoints / rewind** let you snapshot session state and roll back the conversation and edits to an earlier point — the safety net for "Claude went down the wrong path; undo to the last good state" without losing the whole session. **Background tasks** run long-lived commands (servers, watchers, builds) detached so the agent loop continues while they run, re-surfacing output as needed.

**Exam signal (S2):** "Claude made several edits that broke the build; revert to before that without restarting" = checkpoint/rewind. "Keep the dev server running while continuing to work" = background task.

## D3 Anti-Pattern Quick Map

For the exam, bind each Claude Code anti-pattern to its fix: critical rule enforced in a prompt/CLAUDE.md → move to a **hook or permissions deny** (deterministic). Same-session self-review → **fresh-context reviewer subagent**. 18 tools on one agent → **split across subagents / MCP scopes** to ~4–7 each. Personal data committed to the repo → **CLAUDE.local.md / local settings** (gitignored). Relying on `/compact` to retain a conversation-only instruction → **write it into project CLAUDE.md**, which is re-injected after compaction (nested subdirectory CLAUDE.md files are not auto-re-injected). These mappings are the highest-yield items for D3 scenario questions.
