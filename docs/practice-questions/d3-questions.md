# Domain 3 — Claude Code Configuration & Workflows (20%) — Practice Questions

These scenario-based practice questions target **Domain 3 (Claude Code Configuration & Workflows)** of the Claude Certified Architect – Foundations (CCA-F) exam. They are original study items for personal self-study, not live exam content. Topics covered: CLAUDE.md hierarchy/precedence/imports, settings.json and the permissions model, plan mode, custom slash commands, hooks for deterministic enforcement, subagents, MCP servers inside Claude Code, and headless mode for CI/CD. Many items are anchored to scenario **S2 (Code Generation with Claude Code)** and **S5 (Claude Code for CI/CD)**.

Primary sources for verification: Claude Code docs (https://docs.anthropic.com/en/docs/claude-code), the settings and hooks reference, and the Model Context Protocol spec (https://modelcontextprotocol.io). Where an exact value is uncertain the explanation describes behavior conceptually rather than inventing a number.

## Section A — CLAUDE.md Hierarchy, Precedence, and Imports

### Q1. In Claude Code, where does the project-level `CLAUDE.md` file live and what is its primary purpose in a scenario like S2 code generation?

- **A.** In the user's home directory at `~/.claude/CLAUDE.md`, providing machine-wide defaults only
- **B.** At the root of the project repository (checked into version control), providing shared project context and conventions to every contributor's Claude Code session
- **C.** Inside `.git/CLAUDE.md`, where it is hidden from collaborators
- **D.** Only inline in each prompt; CLAUDE.md is a legacy concept no longer read

**Answer:** `B`

**Explanation:** The project `CLAUDE.md` sits at the repo root and is committed so all contributors share the same conventions, build commands, and guardrails — exactly what S2 needs for consistent code generation. Option A describes the user/global memory file (a real but different layer). Tags: D3 · S2 · Foundational.

### Q2. A team wants build commands shared by everyone, plus personal shortcuts that are NOT committed. Which CLAUDE.md layering correctly satisfies both in Claude Code?

- **A.** Put everything in the project `CLAUDE.md` and use `.gitignore` selectively on individual lines
- **B.** Project `CLAUDE.md` (committed) for shared conventions, and `CLAUDE.local.md` or the user-level `~/.claude/CLAUDE.md` for personal, uncommitted preferences
- **C.** Two project `CLAUDE.md` files in the same directory
- **D.** Personal preferences must go in `settings.json` because CLAUDE.md cannot be personalized

**Answer:** `B`

**Explanation:** Shared context belongs in the committed project `CLAUDE.md`; personal, machine-specific notes belong in user-scope memory (`~/.claude/CLAUDE.md`) or a local, git-ignored project memory file. You cannot gitignore individual lines of a tracked file (A). Tags: D3 · S2 · Intermediate.

### Q3. Claude Code reads memory from several locations. When the same instruction conflicts, which precedence model should you assume for the merged context?

- **A.** All files are concatenated; the longest file silently wins
- **B.** More-specific / closer-scoped memory (project, and within it subdirectory) layers on top of broader scope (user/global), with the nearer instruction taking effect for that path
- **C.** The user-global file always overrides the project file
- **D.** Only one file is ever loaded; the rest are ignored

**Answer:** `B`

**Explanation:** Memory is layered from broad to specific — enterprise/user/global provides defaults, and project (and nested subdirectory) memory refines them for that path, with the more specific instruction winning on conflict. The global file does not blanket-override project context (C). Tags: D3 · S2 · Intermediate.

### Q4. Your monorepo has a root `CLAUDE.md` and a `CLAUDE.md` inside `services/payments/`. When Claude Code is working on a file under `services/payments/`, what happens?

- **A.** Only the root file is used; nested CLAUDE.md files are ignored
- **B.** Both apply — the nested `services/payments/CLAUDE.md` adds path-specific context on top of the root file's shared context
- **C.** The nested file replaces the root file entirely for the whole session
- **D.** Nested files cause an error and must be merged manually

**Answer:** `B`

**Explanation:** Claude Code supports nested memory: subtree `CLAUDE.md` files contribute additional context for work in that subtree, layered on top of the root project memory. This is ideal for monorepos where a payments service has stricter rules than the repo default. Tags: D3 · S2 · Intermediate.

### Q5. Which syntax lets one `CLAUDE.md` pull in another file's contents so you can keep shared guidance DRY?

- **A.** `<<include other.md>>`
- **B.** `@path/to/other-file.md` import syntax (an @-reference to the file path)
- **C.** `require('./other.md')`
- **D.** Imports are not supported; you must copy-paste content

**Answer:** `B`

**Explanation:** CLAUDE.md supports `@path` imports, letting you reference another file (e.g., `@docs/coding-standards.md`) so shared standards live in one place and are pulled into memory. This keeps S2 conventions DRY across multiple memory files. Tags: D3 · S2 · Intermediate.

### Q6. What is the fastest in-session way to add a durable instruction to project memory while working in Claude Code?

- **A.** Edit the binary settings database directly
- **B.** Start a line with the `#` shortcut to quickly add a memory, then choose which memory file to save it to
- **C.** Memory can only be edited by restarting the CLI
- **D.** Paste it into the chat; it is auto-saved to CLAUDE.md every turn

**Answer:** `B`

**Explanation:** Typing a line beginning with `#` prompts Claude Code to add that note to a memory file (project or user), making it durable across sessions. Chat messages are not automatically persisted to CLAUDE.md (D). Tags: D3 · S2 · Foundational.

### Q7. A reviewer says the CLAUDE.md is 4,000 lines and Claude "ignores half of it." What is the BEST architectural fix?

- **A.** Keep everything; models read everything perfectly regardless of length
- **B.** Trim CLAUDE.md to concise, high-signal rules and move large reference material into imported files or docs that are pulled in only when relevant
- **C.** Move all rules into the system prompt via an undocumented flag
- **D.** Delete CLAUDE.md entirely and rely on prompting each session

**Answer:** `B`

**Explanation:** Memory competes for context budget; bloated CLAUDE.md dilutes attention. Keep it tight and high-signal, and use `@imports` / on-demand docs for bulk reference. Deleting it (D) loses the shared conventions S2 depends on. Tags: D3 · S2 · Advanced.

### Q8. Which statement about enterprise/managed policy memory in Claude Code is correct?

- **A.** There is no organization-level layer; only user and project exist
- **B.** An enterprise-managed policy layer can set organization-wide guardrails that individual users cannot simply override in their personal memory
- **C.** Enterprise policy is just a renamed project file
- **D.** Enterprise policy only affects billing, not behavior

**Answer:** `B`

**Explanation:** Claude Code supports an enterprise/managed-policy scope so administrators can enforce org-wide rules above user and project memory, which personal files cannot freely override. This is how a regulated org keeps guardrails consistent across teams. Tags: D3 · Advanced.

## Section B — settings.json and the Permissions Model

### Q9. In Claude Code, what is the role of `.claude/settings.json` checked into a repo?

- **A.** It stores your API key in plaintext for the team
- **B.** It defines shared, version-controlled configuration (permissions, hooks, env, model defaults) that applies to everyone using the repo
- **C.** It is a cache file you should delete regularly
- **D.** It only controls terminal colors

**Answer:** `B`

**Explanation:** Project `.claude/settings.json` is committed shared configuration: permission allow/deny rules, hooks, environment, and defaults. Secrets/keys should never be committed there (A). Tags: D3 · S2 · Foundational.

### Q10. You want a permission rule that applies only to YOU on this machine and is never committed. Which file should hold it?

- **A.** `.claude/settings.json`
- **B.** `.claude/settings.local.json` (git-ignored, personal project overrides)
- **C.** `CLAUDE.md`
- **D.** `package.json`

**Answer:** `B`

**Explanation:** `.claude/settings.local.json` holds personal, machine-specific overrides that are not committed, layering above the shared project settings. `CLAUDE.md` is memory/context, not permission configuration. Tags: D3 · S2 · Intermediate.

### Q10b. When the same permission key is defined in multiple settings files, which generally takes precedence?

- **A.** The shared project `.claude/settings.json` always wins
- **B.** More specific / local scope wins: enterprise-managed policy is highest, then command-line args, then local project settings, then shared project, then user settings
- **C.** Whichever file was edited last
- **D.** User-global settings always override everything

**Answer:** `B`

**Explanation:** Settings are layered; managed enterprise policy outranks everything, and among the rest the more local/specific scope (local project) overrides the broader (shared project, then user). This lets orgs enforce non-overridable guardrails. Tags: D3 · Advanced.

### Q11. Claude Code's permission system uses allow/ask/deny lists. What does a `deny` rule guarantee relative to an `allow` rule for the same tool?

- **A.** Allow always wins because it is more permissive
- **B.** Deny takes precedence — a denied action is blocked even if another rule would allow it
- **C.** They cancel out and the user is never prompted
- **D.** Deny only logs a warning but still runs

**Answer:** `B`

**Explanation:** Deny is the strongest outcome: if a rule denies an action it is blocked regardless of any allow rule, which is essential for guardrails (e.g., never run `rm -rf` on prod paths). This is a deterministic control, not a prompt plea. Tags: D3 · S5 · Intermediate.

### Q12. For S5 CI/CD, you run Claude Code non-interactively and want it to act without stopping to ask the human for each tool. Which approach is correct AND safe?

- **A.** Pipe `yes` into stdin to auto-confirm every prompt
- **B.** Pre-configure explicit allow rules (and necessary deny rules) so the needed tools are permitted, scoping permissions narrowly to what the job requires
- **C.** Disable all permission checks globally on the CI runner
- **D.** Parse Claude's output for the word "confirm" and echo back "yes"

**Answer:** `B`

**Explanation:** In automation you grant the specific permissions the job needs up front via allow rules and keep deny guardrails, rather than blanket-disabling safety. Blindly auto-confirming everything (A/C) removes the guardrails that prevent destructive actions in CI. Tags: D3 · S5 · Intermediate.

### Q13. A permission rule like `Bash(npm run test:*)` in the allow list does what?

- **A.** Allows literally only the string `npm run test:*`
- **B.** Allows `npm run test:` followed by any subcommand pattern matched by the rule, while still requiring approval for other Bash commands
- **C.** Allows all Bash commands because it contains a wildcard
- **D.** Denies the test command

**Answer:** `B`

**Explanation:** Permission rules can scope a tool to specific command patterns (e.g., a `test:` prefix), allowing matching invocations without prompting while everything else still follows the ask/deny flow. It does not blanket-allow all Bash (C). Tags: D3 · S5 · Intermediate.

### Q14. Why is committing API keys into `.claude/settings.json` an anti-pattern, and what is the correct alternative?

- **A.** It is fine if the repo is private
- **B.** Secrets in committed config leak to everyone with repo access and into git history; use environment variables / a secrets manager and reference them, never commit raw keys
- **C.** Keys must live in CLAUDE.md instead
- **D.** Claude Code encrypts the file automatically so it is safe

**Answer:** `B`

**Explanation:** Committed secrets are exposed to all collaborators and persist in history. Use environment variables or a secrets manager and inject at runtime; settings can reference env without storing the secret value. Tags: D3 · S5 · Intermediate.

### Q15. Which is the safest default permission posture for a brand-new Claude Code project before you've reviewed what it will do?

- **A.** Allow all tools including arbitrary Bash and file writes outside the repo
- **B.** Start restrictive — let Claude read and propose, require approval (ask) for writes and shell, then progressively add narrow allow rules as you learn the workflow
- **C.** Deny everything so Claude can never act
- **D.** Turn off the permission system to move faster

**Answer:** `B`

**Explanation:** Least-privilege: begin with ask/deny defaults and incrementally grant narrow allows as trust is established. Allow-all (A) and disabling permissions (D) invite destructive actions; deny-everything (C) makes the tool useless. Tags: D3 · Foundational.

## Section C — Plan Mode

### Q16. What does plan mode in Claude Code do, and why is it valuable for an S2 multi-file change?

- **A.** It executes edits immediately but in a sandbox
- **B.** It restricts Claude to read-only investigation and produces a proposed plan for your approval before any file is modified or command run
- **C.** It deletes the working tree to start fresh
- **D.** It is only for image generation

**Answer:** `B`

**Explanation:** Plan mode is a read-only/analysis mode: Claude explores the codebase and presents a plan you approve before it makes changes — ideal for large or risky S2 edits where you want to vet the approach first. It does not mutate files until you accept (A is wrong). Tags: D3 · S2 · Foundational.

### Q17. Which workflow correctly uses plan mode to reduce rework on a complex refactor?

- **A.** Skip planning; let Claude edit and fix mistakes afterward
- **B.** Enter plan mode, have Claude read the relevant code and propose a step-by-step plan, review/adjust the plan, then approve to switch into execution
- **C.** Ask Claude to write the plan into CLAUDE.md permanently
- **D.** Use plan mode only after the edits are done to document them

**Answer:** `B`

**Explanation:** The value of plan mode is front-loading alignment: investigate, propose, review, then execute — catching wrong assumptions before code changes. Documenting after the fact (D) misses the point of preventing rework. Tags: D3 · S2 · Intermediate.

### Q18. During plan mode, you notice Claude's proposed plan touches a file that must never change. What is the deterministic way to guarantee it is never edited — not just "asking nicely"?

- **A.** Add "please don't touch this file" to the prompt and hope it complies
- **B.** Use a permission deny rule (or a hook) to block writes to that path so the protection is enforced by code, not by prompt compliance
- **C.** Rename the file mid-session
- **D.** Trust plan mode to remember across sessions

**Answer:** `B`

**Explanation:** Critical guarantees must be deterministic. A deny permission rule or a hook hard-blocks the write regardless of model behavior. Relying on a prompt plea (A) is anti-pattern #3 — prompt-based enforcement for a critical rule. Tags: D3 · S2 · Advanced.

### Q19. Which statement best distinguishes plan mode from simply asking Claude "make a plan first"?

- **A.** They are identical in every way
- **B.** Plan mode is an enforced mode that actually restricts Claude's ability to make changes until approval, whereas a prompted "plan first" relies on the model voluntarily holding back
- **C.** Asking in the prompt is strictly safer
- **D.** Plan mode cannot read files

**Answer:** `B`

**Explanation:** Plan mode is a system-enforced read-only state; a prompted request to plan is voluntary and can be ignored under pressure. The enforced mode is the deterministic control. Tags: D3 · S2 · Intermediate.

## Section D — Custom Slash Commands

### Q20. Where do you define a project-scoped custom slash command in Claude Code, and how is it invoked?

- **A.** As a function in `settings.json`; invoked by REST call
- **B.** As a Markdown prompt file under `.claude/commands/` (committed); invoked as `/command-name`
- **C.** Only as a shell alias outside Claude
- **D.** Inside CLAUDE.md under a `## Commands` heading

**Answer:** `B`

**Explanation:** Custom slash commands are Markdown files in `.claude/commands/` (project) or `~/.claude/commands/` (personal); the filename becomes `/name`. They package reusable prompts/workflows like an S5 review routine. Tags: D3 · S2 · Foundational.

### Q21. You want a `/fix-issue` command that takes the issue number as input. How do you pass and reference that argument inside the command file?

- **A.** Arguments are not supported in slash commands
- **B.** Invoke `/fix-issue 123` and reference the input in the command body via the arguments placeholder (e.g., `$ARGUMENTS`)
- **C.** Hard-code the number and edit the file each time
- **D.** Pass it as an environment variable only

**Answer:** `B`

**Explanation:** Slash command files support an arguments placeholder (`$ARGUMENTS`, and positional `$1`, `$2`) substituted at call time, so `/fix-issue 123` injects `123` into the prompt template. Hard-coding (C) defeats reuse. Tags: D3 · S2 · Intermediate.

### Q22. A slash command needs to include the current git diff in its prompt. What feature makes this possible inside the command file?

- **A.** Slash commands can run shell commands and embed their output (e.g., via a bash execution directive) into the prompt
- **B.** You must paste the diff manually every time
- **C.** Git is unavailable to slash commands
- **D.** Only MCP can read git

**Answer:** `A`

**Explanation:** Slash command files can execute shell commands and inject their stdout (e.g., `!` bash execution) so the live `git diff` becomes part of the prompt — perfect for an S5 review command. Manual pasting (B) defeats automation. Tags: D3 · S5 · Intermediate.

### Q23. What is the difference between a personal and a project slash command location?

- **A.** There is no difference
- **B.** `~/.claude/commands/` is personal/global to you across all projects; `.claude/commands/` is project-scoped and committed for the whole team
- **C.** Project commands cannot take arguments
- **D.** Personal commands override deny rules

**Answer:** `B`

**Explanation:** Personal commands live in your home `~/.claude/commands/` and follow you everywhere; project commands live in the repo and are shared with collaborators. Both support arguments. Tags: D3 · S2 · Foundational.

### Q24. For S5 CI/CD, you build a `/review-pr` slash command. Why is encapsulating the review steps in a command better than re-typing the prompt each run?

- **A.** It runs faster on the GPU
- **B.** It standardizes the exact review checklist/steps so every PR gets a consistent, repeatable review, and it can be versioned and improved in one place
- **C.** It bypasses the permission system
- **D.** Slash commands disable hooks

**Answer:** `B`

**Explanation:** A committed slash command makes the review deterministic and versioned — every contributor and CI run executes the same vetted steps, improving reliability. It does not bypass permissions or hooks (C/D). Tags: D3 · S5 · Intermediate.

## Section E — Hooks for Deterministic Enforcement (vs Prompt-Pleading Anti-Pattern #3)

### Q25. In Claude Code, what is a hook?

- **A.** A natural-language reminder placed in CLAUDE.md
- **B.** A user-defined shell command that runs automatically at a defined lifecycle event (e.g., before/after a tool call), enabling deterministic, code-enforced behavior
- **C.** A model parameter that increases creativity
- **D.** A type of MCP server

**Answer:** `B`

**Explanation:** Hooks are configured shell commands triggered on lifecycle events (such as PreToolUse, PostToolUse, Stop), giving you deterministic control independent of what the model decides. They are the antidote to anti-pattern #3 (prompt-pleading). Tags: D3 · S5 · Foundational.

### Q26. Your CLAUDE.md says "ALWAYS run the formatter after editing a file," but it's only followed ~70% of the time. What is the correct fix?

- **A.** Make the instruction ALL CAPS and add "this is critical" three times
- **B.** Add a PostToolUse hook that runs the formatter automatically after file edits, so it happens deterministically every time
- **C.** Lower the temperature
- **D.** Tell the model it will be penalized if it forgets

**Answer:** `B`

**Explanation:** Relying on the model to remember a critical step is anti-pattern #3 (prompt-based enforcement). A PostToolUse hook makes formatting happen 100% of the time via code. Shouting in the prompt (A/D) does not give a guarantee. Tags: D3 · S5 · Intermediate.

### Q27. A PreToolUse hook returns a non-zero exit (or a block decision) when Claude tries to run a forbidden command. What is the effect?

- **A.** Nothing — hooks are advisory only
- **B.** The hook can block the tool call and feed the reason back, deterministically preventing the action regardless of the model's intent
- **C.** It crashes the whole CLI
- **D.** It only logs and lets the command run anyway

**Answer:** `B`

**Explanation:** A PreToolUse hook can deny/block a pending tool call (and surface a reason to the model), enforcing policy in code before execution. This is exactly how you stop, e.g., deploys to prod from a feature branch. Tags: D3 · S5 · Advanced.

### Q28. Which task is the BEST fit for a hook rather than a prompt instruction?

- **A.** Choosing a friendly tone in responses
- **B.** Guaranteeing that secrets are never written to a log file by blocking writes that match a secret pattern before they happen
- **C.** Brainstorming feature ideas
- **D.** Summarizing a document

**Answer:** `B`

**Explanation:** Hooks shine for deterministic guarantees on critical/security-sensitive rules (block a write matching a secret pattern). Tone, brainstorming, and summarizing are inherently probabilistic model tasks, not policy enforcement. Tags: D3 · S5 · Intermediate.

### Q29. What information does Claude Code pass to a hook so the hook can make a decision?

- **A.** Nothing; hooks are blind
- **B.** Structured event data (e.g., the tool name and its input parameters) via stdin/environment, which the hook script inspects to allow, block, or modify
- **C.** Only the model's temperature
- **D.** The entire conversation is always emailed to the hook

**Answer:** `B`

**Explanation:** Hooks receive structured context about the event (tool name, inputs, etc.) so the script can deterministically inspect and decide. That is what lets a hook block a `Bash(rm -rf /)` while allowing safe commands. Tags: D3 · S5 · Advanced.

### Q30. A team uses a hook to enforce a business rule. A reviewer suggests "just put the rule in the system prompt instead." Why is the hook still preferable for a CRITICAL rule?

- **A.** System prompts are slower
- **B.** Hooks enforce the rule deterministically in code (100% reliability), while a prompt instruction is probabilistic and can be skipped — for critical business rules you need the deterministic control, not a request
- **C.** Hooks are cheaper per token
- **D.** There is no difference for critical rules

**Answer:** `B`

**Explanation:** This is the core of anti-pattern #3: never rely on prompt pleading for critical business rules. Hooks (or code/permission rules) give deterministic enforcement; the prompt is a best-effort request the model may not honor under pressure. Tags: D3 · S5 · Advanced.

### Q31. Which lifecycle events are appropriate targets for hooks in Claude Code? (Choose the BEST answer.)

- **A.** Only at CLI startup
- **B.** Events such as before a tool runs (PreToolUse), after a tool runs (PostToolUse), and when the agent is about to stop — chosen to match where you need to gate, react to, or validate behavior
- **C.** Only on user logout
- **D.** Hooks can only fire once per day

**Answer:** `B`

**Explanation:** Hooks attach to meaningful lifecycle points (pre/post tool use, stop, and similar) so you can gate actions, run follow-ups (format/lint/test), or validate completion. Picking the right event is how you enforce the right guarantee. Tags: D3 · S5 · Intermediate.

### Q32. In S5, if you must guarantee that `terraform apply` is never run by Claude in CI without a passing plan check, what is the single MOST reliable mechanism?

- **A.** A CLAUDE.md sentence asking Claude to always plan first
- **B.** A PreToolUse hook (and/or deny rule) that blocks `terraform apply` unless a verified plan/approval condition is met
- **C.** A higher max-iteration cap
- **D.** Lowering temperature to 0

**Answer:** `B`

**Explanation:** Deterministic gating belongs in a hook/deny rule that inspects the command and blocks it unless preconditions hold. A CLAUDE.md plea (A) is anti-pattern #3; iteration caps (C) and temperature (D) do not enforce policy. Tags: D3 · S5 · Advanced.

## Section F — Subagents in Claude Code

### Q33. What problem do subagents in Claude Code primarily solve?

- **A.** They make the model faster by lowering precision
- **B.** They isolate a focused task in its own context window with its own tools/prompt, keeping the main session's context clean and reducing tool overload
- **C.** They remove the need for permissions
- **D.** They are only for image tasks

**Answer:** `B`

**Explanation:** Subagents run a scoped task in a separate context with a curated toolset and instructions, preventing the main thread from bloating and avoiding too-many-tools (anti-pattern #8). This is the coordinator/subagent shape from S3 applied inside Claude Code. Tags: D3 · S3 · Intermediate.

### Q34. Where are reusable custom subagents defined in Claude Code?

- **A.** Only inside the API request body
- **B.** As configuration files (e.g., under `.claude/agents/`) specifying the subagent's name, description, system prompt, and the limited set of tools it may use
- **C.** They cannot be customized
- **D.** Inside `package.json`

**Answer:** `B`

**Explanation:** Custom subagents are defined as files (project `.claude/agents/` or user-level) declaring their purpose, instructions, and an allowed tool list — enabling a tight, curated toolset per agent. Tags: D3 · S3 · Intermediate.

### Q35. Why does giving a subagent a tight, curated tool list (≈4–7 tools) beat handing it 18 tools?

- **A.** Fewer tools always means fewer tokens, which is the only reason
- **B.** Too many tools degrade selection accuracy and bloat context; a curated set scoped to the subagent's job improves reliability — split broad capability across multiple subagents instead
- **C.** 18 tools are required for production
- **D.** Tool count has no effect on behavior

**Answer:** `B`

**Explanation:** Anti-pattern #8: overloading one agent with many tools hurts tool-selection accuracy and context efficiency. Curate per subagent and split capability across subagents/MCP boundaries for reliability. Tags: D3 · S3 · Intermediate.

### Q36. A subagent finishes its task. How should its result return to the main agent for the cleanest context management?

- **A.** Dump the subagent's entire raw transcript into the main context
- **B.** Return a concise, structured result (the findings/summary), keeping the subagent's intermediate reasoning out of the main session's context window
- **C.** Email the result to the user
- **D.** Write it only to a log the main agent cannot read

**Answer:** `B`

**Explanation:** The point of a subagent is context isolation: it does the noisy work and hands back a compact result, so the main thread stays lean. Dumping the full transcript (A) defeats the isolation benefit. Tags: D3 · S3 · Advanced.

### Q37. For an S2 task, which is a good use of a subagent versus doing everything in the main session?

- **A.** Trivial one-line edits the main session can do directly
- **B.** A bounded investigation like "search the codebase and report every place this deprecated API is called," where the search noise should stay out of the main context
- **C.** Choosing a variable name
- **D.** Reading a single short file

**Answer:** `B`

**Explanation:** Subagents earn their keep on bounded, noisy, parallelizable work (broad codebase searches/sweeps) whose intermediate output would pollute the main thread. Trivial edits (A/C/D) add coordination overhead for no benefit. Tags: D3 · S2 · S4 · Intermediate.

### Q38. In a coordinator/subagent pattern inside Claude Code, what is a key error-handling rule for the coordinator?

- **A.** Silently drop any subagent that errors and continue as if it succeeded
- **B.** Surface subagent failures explicitly (including diagnostic context) so the coordinator can retry, escalate, or report — never treat an error as empty success
- **C.** Always retry forever with no cap
- **D.** Convert all errors into a generic "something went wrong" with no detail

**Answer:** `B`

**Explanation:** Anti-patterns #6 and #7: hiding diagnostic context and treating errors as empty success are both wrong. The coordinator must see real failures with context to decide retry/escalate. Infinite retry (C) is also unsafe. Tags: D3 · S3 · Advanced.

## Section G — MCP Servers Inside Claude Code

### Q39. What does configuring an MCP server in Claude Code give the agent?

- **A.** A new base model
- **B.** Access to external tools, resources, and prompts exposed by that server (e.g., a database, ticketing system, or docs index) over the Model Context Protocol
- **C.** A way to bypass permissions
- **D.** Faster token streaming only

**Answer:** `B`

**Explanation:** MCP servers expose tools/resources/prompts that Claude Code can use, extending the agent to external systems via a standard protocol (modelcontextprotocol.io). This is how S4 connects Claude to your real services. Tags: D3 · S4 · Foundational.

### Q40. Which file conventionally declares project-shared MCP servers that should be available to everyone on the repo?

- **A.** `.mcp.json` at the project root (committed), so collaborators get the same MCP servers
- **B.** `mcp.log`
- **C.** It can only be done via command-line flags, never a file
- **D.** Inside CLAUDE.md as prose

**Answer:** `A`

**Explanation:** A committed `.mcp.json` declares project-scoped MCP servers so every contributor's Claude Code shares the same integrations. User-scope MCP config is also possible for personal servers. Tags: D3 · S4 · Intermediate.

### Q41. An MCP server can be connected via different transports. Which pairing is correct?

- **A.** MCP only supports email transport
- **B.** Local servers commonly use stdio; remote servers commonly use an HTTP-based transport (e.g., streamable HTTP / SSE)
- **C.** All MCP servers must run in the cloud
- **D.** MCP requires a custom binary protocol with no standard

**Answer:** `B`

**Explanation:** MCP defines transports: stdio for local processes and HTTP-based transports (streamable HTTP / SSE) for remote servers. Choosing the transport depends on where the server runs. Tags: D3 · S4 · Intermediate.

### Q42. A third-party MCP server exposes 30 tools, but your agent only needs 3 of them. What is the best practice in Claude Code?

- **A.** Expose all 30 to the agent; more tools can't hurt
- **B.** Limit the agent to the small subset it actually needs (curate the toolset), to avoid the too-many-tools degradation and keep tool selection accurate
- **C.** Rewrite the MCP server in another language
- **D.** Disable MCP entirely

**Answer:** `B`

**Explanation:** Even with a rich MCP server, hand the agent only the tools it needs — this directly mitigates anti-pattern #8 (too many tools). Curated boundaries keep selection reliable. Tags: D3 · S4 · Intermediate.

### Q43. When you add an untrusted/community MCP server, what is the correct security posture in Claude Code?

- **A.** Trust it fully and grant all permissions automatically
- **B.** Treat its tools as untrusted input: review what it does, scope permissions tightly, and be cautious of prompt-injection via tool results / resources it returns
- **C.** MCP servers cannot pose any security risk
- **D.** Disable your deny rules so the server works smoothly

**Answer:** `B`

**Explanation:** External MCP tools (and the content they return) are an untrusted surface; scope permissions, review the server, and guard against prompt injection from tool outputs. Disabling deny rules (D) removes your guardrails. Tags: D3 · S4 · Advanced.

### Q44. What distinguishes an MCP **resource** from an MCP **tool** in the way Claude Code uses them?

- **A.** They are identical
- **B.** A tool is an action the model can invoke (often with side effects); a resource is contextual data/content the server can provide for the model to read
- **C.** Resources can only be images
- **D.** Tools cannot have parameters

**Answer:** `B`

**Explanation:** In MCP, tools are callable actions (model-invoked, may have effects) while resources are addressable content/data exposed for context. Distinguishing them helps you design the right integration for S4. Tags: D3 · S4 · Intermediate.

## Section H — Headless Mode for CI/CD (S5)

### Q45. What is "headless"/non-interactive Claude Code, and when is it used?

- **A.** Running Claude Code with no model
- **B.** Running Claude Code programmatically without the interactive TUI — typically via the `-p`/print (non-interactive) invocation — so it can be driven by scripts, hooks, and CI pipelines
- **C.** A mode that only prints help text
- **D.** A GUI-only mode

**Answer:** `B`

**Explanation:** Headless mode (e.g., the `-p`/print flag) runs Claude Code non-interactively for automation — the backbone of S5 CI/CD jobs where no human is at the keyboard. Tags: D3 · S5 · Foundational.

### Q46. In S5 CI you need machine-parseable output to gate the build. Which Claude Code option helps most?

- **A.** Rely on regex over the colored TUI output
- **B.** Request structured/JSON output format from the headless run so the pipeline can parse the result deterministically
- **C.** Take a screenshot and OCR it
- **D.** There is no way to get structured output

**Answer:** `B`

**Explanation:** Headless mode can emit structured output (e.g., a JSON output format), giving the pipeline a deterministic, parseable result instead of scraping human-formatted text. This pairs with D4 structured-output discipline. Tags: D3 · S5 · D4 · Intermediate.

### Q47. A CI job invokes headless Claude Code to review a diff and must FAIL the build on a blocking issue. What is the robust signaling mechanism?

- **A.** Look for the phrase "this looks bad" in the prose and grep for it
- **B.** Have the run produce a structured verdict and map it to the process exit code (and/or parse the JSON result), so the pipeline gates on a deterministic signal — not on natural-language phrasing
- **C.** Always pass; humans will catch it later
- **D.** Parse "I'm done" to decide success

**Answer:** `B`

**Explanation:** Gate on deterministic signals (exit code / structured field), never on free-text phrasing. Parsing natural language like "I'm done" to drive control flow is anti-pattern #1; greppy prose matching (A) is brittle. Tags: D3 · S5 · Advanced.

### Q48. For a large S5 multi-pass review across many files, which Claude API capability complements headless Claude Code for throughput and cost?

- **A.** The Batch API, to process many independent review units asynchronously at lower cost
- **B.** Increasing temperature to review faster
- **C.** Removing all tools
- **D.** Disabling prompt caching

**Answer:** `A`

**Explanation:** The Batch API processes many independent requests asynchronously and at reduced cost — a natural fit for fanning out per-file or per-rule review passes in CI. Prompt caching (which you would keep, not disable) further cuts cost on shared context. Tags: D3 · S5 · D5 · Intermediate.

### Q49. In headless CI runs, how should permissions typically be handled compared to interactive use?

- **A.** Identical — always prompt the human
- **B.** Pre-grant the narrow set of tool permissions the job needs (since no human can answer prompts), keeping deny guardrails, rather than relying on interactive approval
- **C.** Disable the permission system entirely
- **D.** Grant all permissions globally on every runner

**Answer:** `B`

**Explanation:** No human is present to answer ask prompts, so you must pre-authorize exactly the tools the job needs and keep deny guardrails. Disabling permissions or granting everything (C/D) removes the safety net in an unattended environment. Tags: D3 · S5 · Intermediate.

### Q50. Why is an arbitrary `--max-turns` style cap the WRONG primary control for when a headless agentic run should stop?

- **A.** It is the correct primary control and should be relied on
- **B.** Iteration caps are a safety backstop, not the control logic — the run should terminate based on real task-completion signals (e.g., the model's stop_reason / a structured "done" verdict), with the cap only as a last-resort guard
- **C.** Caps make the run more accurate
- **D.** Caps replace the need for permissions

**Answer:** `B`

**Explanation:** Anti-patterns #1 and #2: control flow should key off true completion signals (stop_reason / structured result), not natural language, and arbitrary caps are only a backstop. Using the cap as the primary stop hides whether the task actually finished. Tags: D3 · S5 · Advanced.

## Section I — Cross-Cutting Config Reliability (S2/S5)

### Q51. Claude in an S2 session keeps re-running the full test suite when only one file changed, burning time — what is the most maintainable fix?

- **A.** Tell Claude in chat each time to run a narrower command
- **B.** Encode the project's correct, scoped test commands in CLAUDE.md (and/or a slash command), so the right command is the default known behavior
- **C.** Delete the test suite
- **D.** Lower max tokens so it stops sooner

**Answer:** `B`

**Explanation:** Durable, shared config (CLAUDE.md project conventions and/or a `/test` slash command) makes the correct scoped command the default for everyone, instead of re-explaining each session. This is standard S2 memory hygiene. Tags: D3 · S2 · Intermediate.

### Q52. A hook formats files after edits, but it occasionally rewrites files Claude didn't touch, causing noisy diffs — what is the BEST fix?

- **A.** Remove the hook and go back to prompt-pleading
- **B.** Scope the hook to operate only on the files actually edited (use the event's tool-input data) so it stays deterministic but narrowly targeted
- **C.** Accept the noise; hooks can't be scoped
- **D.** Disable git

**Answer:** `B`

**Explanation:** Hooks receive structured event data; use it to target only the edited file(s), keeping the deterministic guarantee while eliminating collateral changes. Reverting to prompt-pleading (A) re-introduces anti-pattern #3. Tags: D3 · S5 · Advanced.

### Q53. You want Claude Code to NEVER read or write a sensitive `.env.production` file. Which combination is strongest?

- **A.** A note in CLAUDE.md saying "avoid this file"
- **B.** A deny permission rule for that path plus, if needed, a PreToolUse hook — deterministic blocking rather than a request the model might overlook
- **C.** Renaming the file each session
- **D.** Hoping the model infers it is sensitive

**Answer:** `B`

**Explanation:** Deterministic protection = deny rule (and/or hook), which enforce regardless of model judgment. A CLAUDE.md note (A) is anti-pattern #3 for a critical security rule. Tags: D3 · S5 · Advanced.

### Q54. In a monorepo where the frontend subtree must use a different package manager command than the backend, what is the cleanest Claude Code configuration?

- **A.** One giant root CLAUDE.md with conditional prose Claude must reason through every time
- **B.** Nested `CLAUDE.md` files in the frontend and backend subtrees, each stating that subtree's correct commands, so context is path-specific automatically
- **C.** Ask the user each time which command to run
- **D.** Put both commands in settings.json permissions

**Answer:** `B`

**Explanation:** Nested memory shines here: each subtree's `CLAUDE.md` declares its own commands and is applied automatically when Claude works in that path — no per-turn reasoning needed. Permissions (D) control allow/deny, not which command is conventional. Tags: D3 · S2 · Intermediate.

### Q55. Which is the correct mental model for the relationship between CLAUDE.md, settings.json, hooks, and permissions?

- **A.** They are interchangeable and you should pick one
- **B.** CLAUDE.md = context/conventions (probabilistic guidance); settings.json = configuration container; permissions = allow/ask/deny gates; hooks = deterministic code-enforced behavior at lifecycle events — use deterministic layers (permissions/hooks) for critical rules and CLAUDE.md for soft guidance
- **C.** Hooks are a kind of CLAUDE.md
- **D.** Permissions are advisory like CLAUDE.md

**Answer:** `B`

**Explanation:** Knowing which layer is deterministic (permissions/hooks) versus probabilistic (CLAUDE.md guidance) is the key exam skill: route critical guarantees to code-enforced layers and conventions to memory. Permissions are enforced gates, not advisory (D). Tags: D3 · S2 · S5 · Advanced.

### Q56. A teammate proposes enforcing "all commits must be signed" purely by adding the rule to CLAUDE.md. What's the issue and better approach?

- **A.** No issue; CLAUDE.md fully guarantees it
- **B.** CLAUDE.md guidance is probabilistic and can be skipped; enforce signing deterministically (e.g., a hook gating the commit, plus repo/CI config), keeping CLAUDE.md only as documentation
- **C.** Just raise the temperature
- **D.** Use a longer prompt with more emphasis

**Answer:** `B`

**Explanation:** Critical process rules need deterministic enforcement (hook + repo/CI policy), not a prompt plea — anti-pattern #3. CLAUDE.md can document the rule but cannot guarantee it. Tags: D3 · S5 · Advanced.

### Q57. For S4 codebase exploration, why are Claude Code's built-in file/search tools often preferable to spinning up an MCP server for the same job?

- **A.** MCP is always slower
- **B.** The built-in read/search/edit tools already cover local codebase exploration with no extra setup; reserve MCP for integrating EXTERNAL systems the built-ins can't reach
- **C.** MCP cannot read code
- **D.** Built-in tools require an internet connection

**Answer:** `B`

**Explanation:** Use the right layer: built-in tools handle local file reading/searching/editing out of the box; add MCP when you need to reach external systems (DB, tickets, internal APIs). Don't add an MCP server to duplicate built-ins. Tags: D3 · S4 · Intermediate.

### Q58. An S5 multi-pass review uses three passes (lint-style, security, architecture). Why run them as separate passes/agents rather than one prompt asking for "all issues"?

- **A.** It is purely cosmetic
- **B.** Separate focused passes give each a clear objective and curated tools, reduce missed issues from a sprawling single prompt, and let a security pass run in a fresh context independent of the author's reasoning
- **C.** One combined prompt is always more reliable
- **D.** Multi-pass disables structured output

**Answer:** `B`

**Explanation:** Focused passes improve recall and tool fit, and a fresh-context review avoids inheriting the author's bias (anti-pattern #9, same-session self-review). A single sprawling prompt tends to surface fewer real issues. Tags: D3 · S5 · Advanced.

### Q59. In a headless S5 review, a config change introduces a regression but the agent reports "no issues" because an internal tool errored and returned empty output. What configuration/design failure does this illustrate?

- **A.** Too few tools
- **B.** Silently suppressing errors / returning empty results as success (anti-patterns #6 and #7) — the tool error hid diagnostic context and was treated as a clean pass
- **C.** Excessive permissions
- **D.** Plan mode was enabled

**Answer:** `B`

**Explanation:** Returning empty output on error and treating it as success is the classic silent-failure trap; the agent never saw the real diagnostic. Tools/hooks must surface errors with context so the run can fail loudly. Tags: D3 · S5 · Advanced.

### Q60. Which end-to-end S5 design BEST embodies deterministic CI/CD with Claude Code?

- **A.** Interactive Claude Code with a human approving each step, parsing prose for "looks good," and an iteration cap as the stop rule
- **B.** Headless run with pre-granted narrow permissions, hooks/deny rules enforcing critical guardrails, structured output mapped to exit codes for gating, Batch API for fan-out, and per-slice (not just aggregate) evaluation
- **C.** Allow-all permissions, one mega-prompt asking for "everything," and trust the model to stop when done
- **D.** Disable hooks and rely entirely on CLAUDE.md pleas

**Answer:** `B`

**Explanation:** The robust design routes critical rules to deterministic layers (hooks/deny), gates on structured signals/exit codes (not prose or arbitrary caps — anti-patterns #1/#2), scales with Batch, and evaluates per slice to avoid masked failures (anti-pattern #10). Option A relies on prose parsing and caps; C/D rely on pleas. Tags: D3 · S5 · Advanced.

### Q61. A custom subagent for S4 is defined but never gets used by the main agent. What configuration detail most commonly causes this?

- **A.** The subagent file is too short
- **B.** A vague or missing `description` of when to use it — the main agent relies on the subagent's description to decide delegation, so it must clearly state the trigger conditions
- **C.** Subagents require a GPU
- **D.** Subagents cannot be invoked automatically

**Answer:** `B`

**Explanation:** Delegation hinges on the subagent's description: if it doesn't clearly state when to invoke this agent, the coordinator won't route to it. A precise, trigger-focused description is the fix. Tags: D3 · S4 · S3 · Intermediate.

### Q62. Which is the correct way to make a slash command run a deterministic check and refuse to proceed if it fails?

- **A.** Put "only continue if tests pass" in the command prose and trust the model
- **B.** Have the command execute the check via shell and combine it with a hook/permission gate, so a failing check deterministically blocks the next action rather than depending on the model's judgment
- **C.** Slash commands cannot run checks
- **D.** Use a higher iteration cap to retry until it passes

**Answer:** `B`

**Explanation:** Slash commands can run shell checks, but the hard "refuse to proceed" guarantee comes from a hook/permission gate. Trusting prose to enforce a gate is anti-pattern #3; bumping the cap (D) doesn't enforce anything. Tags: D3 · S5 · Advanced.

### Q63. A team reports per-language quality varies wildly, but their single CI gate reports one overall "pass rate" that looks fine. Which D3/S5 design change addresses this?

- **A.** Raise the overall threshold
- **B.** Evaluate and gate per slice (per language / per file-type) instead of only on an aggregate metric, so a failing slice can't be masked by strong slices
- **C.** Reduce the number of files reviewed
- **D.** Remove the failing language from CI

**Answer:** `B`

**Explanation:** Anti-pattern #10: aggregate metrics mask per-slice failures. Break the gate down by language/file-type so weak slices surface. Removing the failing language (D) is a workaround that hides the problem, not a fix. Tags: D3 · S5 · Advanced.

### Q64. What is the safest way to let Claude Code use a personal MCP server (e.g., your private notes index) without exposing it to teammates?

- **A.** Commit it in `.mcp.json`
- **B.** Configure it in your user-scope MCP configuration (not the committed project `.mcp.json`), so only your sessions load it
- **C.** Paste the server URL into CLAUDE.md
- **D.** There is no way to keep an MCP server personal

**Answer:** `B`

**Explanation:** Project `.mcp.json` is shared/committed; for a personal server use user-scope MCP config so only you load it. CLAUDE.md is context, not an MCP transport config (C). Tags: D3 · S4 · Intermediate.

### Q65. Putting it together for S2: which combination gives a new contributor a consistent, safe Claude Code experience on day one?

- **A.** No shared config; everyone configures their own machine ad hoc
- **B.** Committed project `CLAUDE.md` (conventions + imports), `.claude/settings.json` (permissions, hooks), `.claude/commands/` (shared workflows), and `.mcp.json` (shared integrations) — all version-controlled
- **C.** Only a long onboarding doc in a wiki
- **D.** Allow-all permissions plus a verbal explanation

**Answer:** `B`

**Explanation:** Version-controlled CLAUDE.md, settings.json (permissions/hooks), shared slash commands, and `.mcp.json` make the whole workflow reproducible and safe for every contributor from the first session — the essence of S2 configuration. Tags: D3 · S2 · Intermediate.
