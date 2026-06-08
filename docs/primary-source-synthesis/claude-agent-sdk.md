# Claude Agent SDK — Architecture, Agent Loop, and Configuration

The **Claude Agent SDK** (formerly the *Claude Code SDK*) packages the same agent loop, built-in tools, context management, and permission system that power Claude Code, and exposes them as a programmable library in **TypeScript** (`@anthropic-ai/claude-agent-sdk`) and **Python** (`claude-agent-sdk`). It is the recommended foundation for building production agents on top of Claude. Official reference: the Agent SDK overview at platform.claude.com/docs (`/en/agent-sdk/overview`).

The single most important framing for the exam: the Agent SDK *brings the harness*. The raw **Messages API** gives you one model turn at a time — you implement the tool-execution loop, context window management, and permission gating yourself. The Agent SDK runs that loop for you. Choosing the SDK over hand-rolling a Messages-API loop is the default-correct architecture answer whenever a scenario describes an autonomous, multi-step, tool-using agent.

> Exam signal (D1, S1, S3): When a scenario asks "what is the cleanest way to build an agent that reads files, runs commands, and verifies its own work across many turns," the answer is the Agent SDK, *not* a custom `while stop_reason == "tool_use"` loop on the Messages API. Hand-rolling the loop is the distractor that re-implements what the SDK already provides.

## The Agent Loop: Gather Context, Take Action, Verify Work, Repeat

The SDK implements Claude Code's core loop: **gather context → take action → verify work → repeat**. Claude pulls in relevant context (reads files, searches the codebase, fetches web pages), takes an action (edits a file, runs a command, calls a tool), verifies the result (re-reads output, runs tests, checks for errors), and repeats until the task is complete. The SDK manages the conversation, executes tool calls, feeds results back to the model, and decides when to continue versus stop — all without you writing loop control.

Crucially, **the loop terminates on the model's `stop_reason`, not on parsed natural language.** The SDK ends a turn when Claude stops requesting tools (the API reports it is done), surfacing a final result message. You should never gate completion on scanning the assistant text for phrases like "I'm done" or "task complete."

> Exam signal (D1, S3 — anti-pattern #1 and #2): "Parse the response for 'I'm finished' to decide whether to loop again" is always wrong; rely on the API stop signal the SDK already checks. An arbitrary iteration cap (e.g. `max_turns=10`) is a *safety backstop*, not the primary control — making the cap the main stopping mechanism is anti-pattern #2.

## query() One-Shot vs ClaudeSDKClient Persistent Session

The SDK exposes two entry points with different lifecycle semantics.

**`query()`** is the one-shot, streaming entry point. You pass a `prompt` and `options`, then iterate the async stream of messages it yields (system init message, assistant messages, tool-use/tool-result messages, and a final result message). Each `query()` call is a fresh interaction unless you explicitly resume a prior session. Use it for single tasks, CI jobs, and stateless request/response work.

```python
# Python
async for message in query(
    prompt="Find and fix the bug in auth.py",
    options=ClaudeAgentOptions(allowed_tools=["Read", "Edit", "Bash"]),
):
    print(message)
```

```typescript
// TypeScript
for await (const message of query({
  prompt: "Find and fix the bug in auth.ts",
  options: { allowedTools: ["Read", "Edit", "Bash"] }
})) {
  console.log(message);
}
```

**A persistent client/session** (the `ClaudeSDKClient` pattern in Python; the streaming-input client form in TypeScript) keeps a long-lived connection for *multi-turn, interactive* agents. You send messages over the life of the client and receive responses while the conversation context, files read, and analysis stay live in the same session. Reach for the persistent client when you need bidirectional, turn-by-turn interaction (a chat-style support agent, an interactive coding assistant) rather than firing a single autonomous task.

> Exam signal (S1): A customer-support agent that holds a conversation with a user across several messages is a persistent-session use case. A batch job that resolves one ticket end-to-end and exits is a `query()` use case. The exam tests whether you pick the right lifecycle for the interaction pattern.

## Configuring Options: System Prompt, Tools, Permission Mode, Working Directory

Configuration flows through an options object — `ClaudeAgentOptions` in Python, the `options` object in TypeScript. Field names differ by language convention: Python uses `snake_case` (`allowed_tools`), TypeScript uses `camelCase` (`allowedTools`).

**System prompt.** Set the agent's instructions/persona via the system-prompt option. You can supply your own string, or extend Claude Code's default system prompt. This is where you define role and domain behavior — but *not* where you enforce hard business rules (see hooks below).

**Allowed and disallowed tools.** `allowed_tools` / `allowedTools` pre-approves a curated set of tools so the agent can use them without prompting; `disallowed_tools` / `disallowedTools` blocks tools outright. A read-only analysis agent might allow only `Read`, `Glob`, `Grep`. Keep the set tight.

> Exam signal (D2 — anti-pattern #8): Granting one agent 18 tools is the wrong answer. Curate roughly 4–7 tools per agent and split broader capability across **subagents** or **MCP-server boundaries**. A bloated tool list degrades selection accuracy and is a classic distractor.

**Permission mode.** The `permission_mode` / `permissionMode` option governs how tool calls are authorized. Common modes include `default` (prompt/approval flow for sensitive actions), `acceptEdits` (auto-accept file edits), `plan` (read/plan without executing mutating actions), and a fully bypassing mode for trusted automation. Use `plan` mode when you want Claude to investigate and propose before touching anything; use `acceptEdits` for unattended pipelines where edits are expected.

**Working directory.** Set `cwd` to scope the agent's filesystem operations to a specific project root. Built-in file tools (`Read`, `Write`, `Edit`, `Glob`, `Grep`) operate relative to this directory.

## Registering MCP Servers

The SDK connects to external systems via the **Model Context Protocol (MCP)** — databases, browsers, internal APIs, SaaS tools, and the broad ecosystem of community servers (modelcontextprotocol.io). Register servers through the `mcp_servers` / `mcpServers` option, keyed by a server name, each specifying how to launch or reach it (e.g. a `command` + `args` for a stdio server, or HTTP/SSE transport config).

```python
options=ClaudeAgentOptions(
    mcp_servers={"playwright": {"command": "npx", "args": ["@playwright/mcp@latest"]}}
)
```

MCP tools surface to the agent alongside built-in tools and are namespaced by server, which is exactly how you keep tool counts manageable: group related external capabilities behind one server boundary rather than registering dozens of loose tools.

> Exam signal (D2, S1, S4): MCP is the right integration layer for "let the agent query our ticketing system / database / browser." Curating which MCP servers (and which of their tools) an agent can see is a tool-design decision, not a prompt decision.

## In-Process Tools (Custom SDK Tools)

Beyond built-in and MCP tools, you can define **in-process custom tools** — TypeScript or Python functions that run inside your own process and are exposed to Claude as callable tools. This is the SDK-native analog of the Messages-API `tool_use` flow, but you don't manage the request loop: you write the function, register it, and the SDK invokes it when Claude calls it and returns the result automatically. In-process tools are ideal for logic that lives in your application (custom calculations, internal service calls) without standing up a separate MCP server.

A well-designed tool returns rich, structured results *including diagnostic context on failure*. Returning a generic "an error occurred" string, or returning empty results on failure as if the call succeeded, blinds the model and produces silent wrong answers.

> Exam signal (D2 — anti-patterns #6 and #7): Tools must surface real error detail (status codes, messages, what was attempted) so Claude can recover or escalate. Silently swallowing an exception and returning `[]` as if successful is always the wrong tool design.

## Hooks: Deterministic Control at Lifecycle Points

**Hooks** run your code at defined points in the agent lifecycle: `PreToolUse`, `PostToolUse`, `Stop`, `SessionStart`, `SessionEnd`, `UserPromptSubmit`, and more. In the SDK, hooks are callback functions (registered via `HookMatcher` in Python / matcher objects in TypeScript) that can **validate, log, block, or transform** behavior. A `PreToolUse` hook can deterministically deny a dangerous command; a `PostToolUse` hook can audit every file write.

Hooks are the correct place to enforce **critical business rules**, because they are deterministic code that the model cannot reason its way around. Asking the model nicely in the system prompt ("never delete production data," "always require approval over $500") is probabilistic and bypassable.

> Exam signal (D1, S1 — anti-pattern #3): When a rule *must* hold (refund caps, PII redaction, approval gates), enforce it with a hook (or in-code check), not with prompt instructions. "Add a strong sentence to the system prompt" is the distractor; "add a PreToolUse hook that blocks the action" is correct.

## Subagents: Delegating Focused Work in Isolated Context

The SDK supports **subagents** — specialized agents your main (coordinator) agent spawns to handle focused subtasks, each with its own instructions, its own curated tool set, and its own context window. Define them via the `agents` option (Python `AgentDefinition`, TypeScript inline object with `description`, `prompt`, and `tools`). Subagents are invoked through the `Agent` tool, so include `Agent` in the coordinator's allowed tools to auto-approve delegation. Messages produced inside a subagent carry a `parent_tool_use_id` so you can trace which delegation they belong to.

Subagents solve two architecture problems at once: they keep each agent's tool list tight (split 18 tools into several 5-tool specialists), and they isolate context so a research subagent's noisy intermediate work doesn't pollute the coordinator's window. They are also the right primitive for **fresh-context review**.

> Exam signal (D1, S3 — anti-pattern #9): A reviewer agent must run in a *separate context* from the author. Reviewing in the same session inherits the author's reasoning bias and tends to rubber-stamp. A coordinator + independent reviewer subagent (or a separate `query()` with a clean session) is the correct multi-agent pattern. This is the canonical S3/S5 "multi-pass review" answer.

## Structured Outputs (Output Schema)

The SDK supports **structured outputs**: you specify an output schema so the agent's final result conforms to a defined JSON shape, rather than free-form prose you have to parse. This is the reliable path for machine-consumable results — extraction pipelines, CI gates, downstream automation. Combine it with a **validation-retry** loop: validate the structured result against your schema, and if it fails, feed the validation error back so the agent corrects it.

> Exam signal (D4, S5, S6): For "the agent's output is consumed by another system," choose schema-constrained structured output plus validate-and-retry. Regexing free text or trusting an unconstrained string is the wrong answer. Retry must pass the *specific* validation error back, not a generic "try again."

## Session Management, Resuming, and Forking

Sessions persist conversation state — files read, analysis done, history — to disk (JSONL on your filesystem). Capture the `session_id` from the system `init` message of a `query()`, then continue later by passing it to the **`resume`** option; the new query starts with the full prior context. You can also **fork** a session to branch and explore alternative approaches from a shared starting point without disturbing the original thread.

```python
# capture from the init message, then:
async for message in query(prompt="Now find all callers",
                           options=ClaudeAgentOptions(resume=session_id)):
    ...
```

> Exam signal (D5): Resume/fork is how the SDK handles long-running and multi-stage work and how you recover or branch deterministically. "Stuff the entire prior transcript back into a new prompt manually" is the distractor; resume the session instead.

## Streaming and Message Types

Both entry points are **streaming-first**: you iterate an async stream of typed messages — a system `init` message (carries `session_id` and config), assistant messages, tool-use and tool-result messages, and a final result message that carries the answer (and, with a schema, the structured payload). Streaming lets you surface progress, log tool activity, and react mid-run (e.g., from a hook). In TypeScript you discriminate on `message.type` / presence of `result`; in Python you check the message class (`SystemMessage`, `ResultMessage`, etc.).

## Filesystem Configuration and settingSources

The SDK can load Claude Code's filesystem configuration — **CLAUDE.md** memory, `.claude/skills/`, `.claude/commands/`, and plugins — from the working directory and `~/.claude/`. Control which sources load with `setting_sources` / `settingSources`. This is what lets an SDK-built agent inherit the same project conventions a human gets from Claude Code, and it ties D3 (Claude Code configuration) directly into SDK-built agents.

## How It Differs from the Raw Messages API (Summary)

| Concern | Raw Messages API | Claude Agent SDK |
| --- | --- | --- |
| Tool-execution loop | You write it (`while stop_reason == "tool_use"`) | Built in |
| Built-in tools (Read/Edit/Bash/Grep…) | You implement each | Provided |
| Context/window management | Manual | Managed by the harness |
| Permissions / approval gating | You build it | `permission_mode` + hooks |
| Sessions / resume / fork | You persist transcripts yourself | First-class (`resume`, fork) |
| Subagents, MCP wiring, structured output | DIY | First-class options |

The Messages API is the right primitive for a single, tightly-controlled model call. The Agent SDK is the right primitive for an autonomous, multi-step, tool-using agent — and for production agents on your own infrastructure, with **Managed Agents** as the hosted alternative when you don't want to operate the sandbox/session infrastructure yourself.

> Exam signal (D1): "We need full control of one model call and will execute tools ourselves" → Messages API. "We need an autonomous agent loop with tools, permissions, and sessions" → Agent SDK. "We want Anthropic to host the agent loop and sandbox" → Managed Agents. Mapping the requirement to the right layer is a recurring D1 question shape.
