# Scenario 4: Developer Productivity with Claude (S4) — Deep Dive

## Scenario 4 Overview: Developer Productivity with Claude Code

S4 puts you in the seat of a solution architect helping a developer (or a whole team) get more done inside an existing codebase using Claude Code. The recurring theme is **tool surface design**: which capability does the job — a built-in tool, an MCP server, or a slash command — and how do you keep the agent's tool set tight, scoped, and least-privilege. Questions are scenario-anchored: "A developer wants Claude to explore a large monorepo and answer questions about it. What is the right configuration?" The trap answers almost always involve over-provisioning (18 tools, broad credentials, redundant MCP servers that duplicate built-ins).

**Exam signal:** When a question describes adding capabilities to Claude Code, the correct answer is the *smallest* surface that satisfies the need. If two options both work, prefer the one with fewer tools and narrower credentials. "It works" is not the bar; "least privilege that works" is.

## Built-in Tools for Codebase Exploration (Read, Edit, Grep, Glob, Bash)

Claude Code ships with a curated set of built-in tools optimized for software work. The core exploration primitives are **Read** (read a file, including images/PDFs/notebooks), **Edit** (exact string replacement in a file), **Grep** (ripgrep-backed content search across the tree), **Glob** (fast path/filename pattern matching), and **Bash** (run shell commands). These are the default vocabulary for understanding and modifying a repo, and they are already fast, sandbox-aware, and permission-gated.

The single most important S4 instinct: **for reading and navigating a local codebase, the built-in tools are the answer.** You do not need an MCP server to read files, search code, or run a build — Read/Grep/Glob/Bash already do this natively and are tuned for it. Reaching for an MCP "filesystem" or "code search" server to do what Grep/Glob already do is the textbook redundant-tool mistake.

Grep is for *content* ("where is `parseConfig` called?"); Glob is for *paths* ("all `*.test.ts` under `src/`"). Bash is for *actions with side effects or tooling* (run tests, `git log`, `npm run build`, `gh` CLI calls). Read is for *pulling specific content into context*. Knowing which primitive maps to which intent is itself testable.

**Exam signal:** If an answer proposes installing an MCP server whose tools overlap built-ins (file read, text search, shell), it is almost certainly the distractor. The official guidance is to use built-ins for local file/search/shell work and reserve MCP for *external systems* the built-ins cannot reach. See docs.anthropic.com/en/docs/claude-code.

## When to Use an MCP Server vs a Built-in Tool vs a Slash Command

Use this decision tree on every S4 "how should we wire this up" question:

1. **Is it reading/searching/editing local files or running local commands?** Use a **built-in tool** (Read/Edit/Grep/Glob/Bash). No MCP needed.
2. **Is it a repeatable prompt/workflow you want to invoke by name (a canned procedure, with the same instructions each time)?** Use a **slash command** (a `.md` file under `.claude/commands/`). Slash commands are prompt templates, not new capabilities — they orchestrate existing tools.
3. **Does it require reaching an external system the built-ins can't touch** — GitHub's API, a database, a ticketing system, an internal docs service, a browser? Use an **MCP server**, scoped and least-privilege.
4. **Is it a specialized, context-heavy exploration you want isolated from the main thread?** Use a **subagent** (see below) — orchestration, not a new tool type.

The clean mental model: **built-ins = local capability; slash commands = reusable prompts/workflows; MCP servers = external capability; subagents = isolated context for a sub-task.** They are not interchangeable, and the wrong answers in S4 deliberately blur these lines (e.g., "write a slash command to read the database" — a slash command can't reach a DB; that needs an MCP server).

**Exam signal:** A question that offers "MCP server," "slash command," and "built-in tool" as options for the *same* need is testing this mapping. Match the need to the right layer. External system → MCP. Reusable prompt → slash command. Local file/shell → built-in.

## Adding MCP Servers at the Right Scope with Least Privilege

MCP (Model Context Protocol, modelcontextprotocol.io) lets Claude Code connect to external tools and data through a standard server interface. In Claude Code you add servers with `claude mcp add` and they can be registered at different **scopes**: **local** (just you, this project), **project** (shared with the team via a checked-in `.mcp.json`), or **user/global** (available across all your projects). Choosing scope is a real exam point.

Scope rule of thumb: a server that the whole team needs for *this* repo (e.g., the project's database, its issue tracker) belongs in **project** scope so it's version-controlled in `.mcp.json` and consistent for everyone. A server that's personal to your workflow across many repos goes in **user** scope. Secrets must never be hard-coded into a checked-in `.mcp.json` — reference environment variables instead so credentials stay out of source control.

**Least privilege is the governing principle.** When you add a GitHub MCP server, give it a token scoped to the specific repos and the minimal permissions the task needs (read-only if the agent only explores; no `admin`/`delete`/org-wide scope if it just reads PRs). When you add a database MCP server, point it at a **read-only** connection/role for exploration work, not a credential that can drop tables. A token that "can do everything" is the wrong answer even when it technically works.

**Exam signal:** Distractors offer broad credentials ("a personal access token with full repo + admin + org scope," "the production DB superuser connection string"). The right answer scopes the token/role to exactly the repos and operations needed and prefers read-only for read-only work. Broad credentials = automatic wrong answer.

## Anti-Pattern #8: Tool Overload — Curate a Tight Tool Set (~4–7)

Loading too many tools onto a single agent degrades performance. With 18 tools available, the model spends reasoning budget deciding *which* tool to use, tool descriptions crowd the context window, and similar tools (three different "search" tools) cause selection errors and inconsistent behavior. The documented best practice is a **tight, curated set — roughly 4–7 tools** that map cleanly to the task, with no two tools doing the same job.

When you genuinely need broader capability, **don't pile more tools onto one agent — split along boundaries.** Push specialized capability into a **subagent** (which has its own tool set and its own context) or behind an **MCP server boundary** (so the parent agent sees a small, purposeful interface rather than a sprawl). A coordinator with 5 tools that delegates DB work to a subagent with its own 4 DB tools is far healthier than one agent holding 9 tools.

Concretely for S4: a developer exploring a codebase usually needs Read, Grep, Glob, Bash, plus maybe one or two external MCP tools (GitHub PR/issue access, a docs lookup). That's the tight set. If a proposed configuration enumerates a dozen-plus tools — especially several that duplicate each other or duplicate the built-ins — it is the anti-pattern the exam wants you to reject.

**Exam signal:** "The team adds 18 tools so Claude can handle anything" is the classic wrong answer (anti-pattern #8). The right answer curates ~4–7, removes redundant/overlapping tools, and offloads specialized work to subagents or MCP boundaries.

## Avoiding Redundant MCP Tools That Duplicate Built-ins

A specific, high-frequency S4 trap: installing MCP servers whose tools re-implement what Claude Code already has. A "filesystem" MCP server duplicates Read/Edit/Glob. A "grep"/"code-search" MCP server duplicates Grep. A "shell" MCP server duplicates Bash. Adding these doesn't add capability — it adds ambiguity (now there are two ways to read a file), bloats the tool list (feeding anti-pattern #8), and can introduce a less-permissioned or slower path than the native one.

The correct posture: **built-ins own local file/search/shell; MCP earns its place only by reaching something built-ins can't** (remote APIs, databases, SaaS, browsers, internal services). Before adding any MCP tool, ask: "Does a built-in already do this?" If yes, don't add the MCP tool. This single check eliminates most over-provisioning.

**Exam signal:** An option that "adds a filesystem MCP server so Claude can read files" is wrong twice over — Claude already reads files via Read, and it inflates the tool set. Prefer the built-in.

## CLAUDE.md: Capturing Project Knowledge So Context Is Reused

`CLAUDE.md` is the project memory file Claude Code automatically loads into context at the start of a session (and merges from multiple levels — enterprise, project-root, and user/global `~/.claude/CLAUDE.md`). It is where you encode durable project knowledge so it doesn't have to be re-discovered or re-explained every session: architecture overview, key directories, build/test/lint commands, coding conventions, "gotchas," and pointers to important files.

In S4, CLAUDE.md is the productivity multiplier. Instead of the model re-exploring the repo from scratch each time (burning tokens and time on Grep/Glob sweeps), a well-maintained CLAUDE.md gives it the map up front: "tests run with `make test`," "domain models live in `core/models/`," "never edit generated files in `gen/`." This turns repeated discovery into reused, cached context. You can scaffold it with `/init`, which inspects the repo and drafts a starting CLAUDE.md.

Keep CLAUDE.md tight and high-signal — it's loaded every session, so bloated or stale content wastes the very context budget it's meant to save. Capture stable facts (commands, layout, conventions), not transient task notes.

**Exam signal:** When a scenario says "Claude keeps re-discovering the same project facts" or "the team re-explains build commands every session," the fix is CLAUDE.md (capture knowledge for reuse), not a bigger model or more tools. See docs.anthropic.com/en/docs/claude-code/memory.

## Subagents for Specialized Exploration Tasks

Subagents let the main (coordinator) thread delegate a focused job to a separate agent that runs in its **own context window** with its **own tool set**. For developer productivity, this is the right tool when an exploration is *noisy* or *context-hungry*: trawling logs, doing a broad grep sweep across a huge monorepo, surveying many files to answer one question. The subagent does the messy work and returns only the distilled findings, keeping the parent's context clean and focused.

Two payoffs land directly on exam objectives. First, **context hygiene** (D5): the verbose intermediate output (hundreds of grep hits, full file dumps) stays in the subagent and never pollutes the main thread — only the summary returns. Second, **tool-surface control** (anti-pattern #8): the subagent carries the specialized tools for its job, so the coordinator's tool list stays small. A "codebase-explorer" subagent with Read/Grep/Glob is a clean way to give deep search power without bloating the main agent.

**Exam signal:** "How do you let Claude do a wide, noisy codebase search without blowing up the main context / without loading 12 tools?" → spawn a subagent for the exploration and keep only its findings. Subagents = isolated context + scoped tools, not just parallelism.

## Slash Commands for Repeatable Developer Workflows

Slash commands are reusable prompt templates stored as Markdown files (project-level in `.claude/commands/`, personal in `~/.claude/commands/`). They are how you make a multi-step procedure invokable by name — `/review-pr`, `/write-tests`, `/explain-module` — so the team runs the same well-crafted instructions every time instead of re-typing them. They can take arguments and embed the steps the workflow should follow.

The key distinction the exam tests: **a slash command does not grant new capability — it packages a prompt that drives the tools the agent already has.** `/run-tests` is valuable because it standardizes *how* you ask Claude to test, but the actual capability comes from Bash. A slash command cannot, by itself, reach a database or call an API; that capability has to come from a built-in or an MCP server. Slash commands orchestrate; they don't connect.

**Exam signal:** "We need a repeatable, named workflow the whole team runs the same way" → slash command. "We need to reach an external system" → MCP server. Don't confuse the two; a slash command is a prompt, not a connector.

## Decision Table: Need → Right Mechanism

| Developer need | Right mechanism | Why / trap to avoid |
|---|---|---|
| Read, search, or edit local files | Built-in Read/Edit/Grep/Glob | Don't add a filesystem/search MCP server — redundant (anti-pattern #8) |
| Run tests, builds, git/gh, lint | Built-in Bash | Don't add a "shell" MCP server — duplicates Bash |
| Reach GitHub PRs/issues/Actions via API | GitHub MCP server, repo-scoped token, read-only if read-only work | Broad full-scope PAT = wrong answer (least privilege) |
| Query a project/internal database | Database MCP server, read-only role/connection | Production superuser credentials = wrong answer |
| Look up internal/library docs | Docs MCP server (or built-in WebFetch if public) | Only if built-ins can't reach it |
| Repeatable named team workflow | Slash command in `.claude/commands/` | A slash command can't reach external systems |
| Persist project facts across sessions | CLAUDE.md (scaffold with `/init`) | Don't rely on re-discovery each session |
| Wide, noisy exploration of a big repo | Subagent (own context + scoped tools) | Keeps main context clean; keeps tool list tight |
| "Handle anything" by loading every tool | NO — curate ~4–7, split via subagents/MCP | Loading 18 tools is anti-pattern #8 |

## S4 Wrong-Answer Catalog (Memorize These Distractors)

**Loading 18 tools "so Claude can do everything."** Wrong — degrades tool selection and bloats context. Curate ~4–7 and split via subagents/MCP boundaries (anti-pattern #8).

**Adding a filesystem / grep / shell MCP server.** Wrong — duplicates Read/Edit/Glob/Grep/Bash. Built-ins own local work; MCP is for external systems only.

**Using a full-scope GitHub PAT or production DB superuser credential.** Wrong — violates least privilege. Scope the token to the needed repos/permissions and use read-only roles/connections for read-only work.

**Hard-coding secrets into a checked-in `.mcp.json`.** Wrong — leaks credentials into source control. Reference environment variables; pick project vs user scope deliberately.

**Writing a slash command to "connect to the database / call the API."** Wrong — slash commands are prompts, not connectors. External access needs an MCP server (or a built-in where applicable).

**Letting Claude re-explore the repo every session instead of writing CLAUDE.md.** Wrong — wastes context and time. Capture stable project knowledge once for reuse.

**Exam signal:** Across S4, the right answer is consistently the *minimal, least-privilege, non-redundant* configuration. When in doubt, pick the option with the fewest tools, the narrowest credentials, and the clearest layer mapping (built-in for local, MCP for external, slash command for reusable prompts, subagent for isolated context).
