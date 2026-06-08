# Domain 2 — Tool Design & MCP Integration (18%)

This domain tests whether you can design tools that Claude *selects correctly and uses reliably*, and whether you understand the Model Context Protocol (MCP) well enough to choose the right architecture, transport, and security posture. Questions are anchored to **S1 (Customer Support Resolution Agent)**, **S4 (Developer Productivity)**, and **S6 (Structured Data Extraction)**. The recurring theme: a tool is a *contract and an interface for the model*, not just a function call — its name, description, schema, and error semantics are all prompt surface.

Primary sources: Anthropic tool-use guide (https://docs.claude.com/en/docs/build-with-claude/tool-use), "Writing tools for agents" (Anthropic engineering), MCP spec (https://modelcontextprotocol.io), and Claude Code MCP docs (https://docs.anthropic.com/en/docs/claude-code/mcp).

## Designing Reliable Tool Schemas: Names, Descriptions, input_schema

A tool definition has three job-critical parts: a **name**, a **description**, and an **input_schema** (JSON Schema). The model decides whether and how to call a tool almost entirely from these. The description is the single highest-leverage field — invest tokens there. Treat it as a mini system prompt for that tool: what it does, *when to use it and when not to*, what each parameter means, units and formats, side effects, and what it returns.

Name tools after the action and domain, unambiguously: `get_customer_order_status`, not `lookup` or `helper`. Names that overlap in meaning (`search` vs `find` vs `query` on the same data) cause the model to pick wrong or oscillate. Distinct, non-overlapping names reduce selection error.

The `input_schema` should be as **precise and constrained** as the real contract: use `enum` for fixed choices, `required` for mandatory fields, `format`/`pattern` for dates and IDs, `description` on every property, and avoid free-form `string` where a structured shape is possible. A tight schema is both better documentation for the model and a validation gate before your code runs. Provide examples in the description for any non-obvious argument.

Exam signal: A stem shows a tool whose description is one terse line ("Gets data") and the model keeps mis-calling it or skipping it. The correct fix is **enrich the description and tighten input_schema** (add when-to-use guidance, parameter semantics, enums) — *not* adding more instructions to the system prompt or lowering temperature.

## Error Semantics: is_error and DIAGNOSTIC Messages (not generic errors)

When a tool fails, return the result as a `tool_result` content block with `is_error: true` and a **diagnostic, actionable message** the model can reason about. Claude reads tool results as conversation input; a useful error lets it self-correct (fix an argument, try a different value, escalate). Examples of good errors: "ZIP code 9410 is invalid: expected 5 digits; received 4," or "Order ID not found; valid IDs start with 'ORD-'. Did you mean to search by email?"

Two named anti-patterns live here. **Anti-pattern #6 — generic error messages** ("An error occurred") strip the diagnostic context the model needs; it cannot recover and either loops or gives up. **Anti-pattern #7 — silently suppressing errors** (catching an exception and returning `[]` or an empty object *without* `is_error`) is worse: the model treats failure as a successful empty result, producing confidently wrong answers. Never mask a failure as success.

Errors should fail *loud and informative* to the model but *safe* to the user: include the diagnostic detail the model needs to retry, but do not leak secrets, stack traces with credentials, or internal infrastructure in user-facing surfaces.

Exam signal: S6 extraction tool returns `{}` when the document is malformed, and the pipeline reports success. The right answer sets `is_error: true` with a message naming the missing/invalid field so the validation-retry loop can act; the wrong answers either keep suppressing or add a try/except that returns empty.

## Right-Sizing the Tool Set: ~4–7 Curated Tools (Anti-pattern #8)

Tool selection accuracy *degrades* as the tool count grows: more options mean more chances to pick the wrong one, more description tokens competing for attention, and more overlap. The exam's canonical bad design is **one agent with ~18 tools** (anti-pattern #8). The right pattern is a **tight, curated set of roughly 4–7 tools** per agent, each with a clear, non-overlapping responsibility.

When you genuinely need many capabilities, you don't cram them into one agent — you **split responsibilities** across boundaries: group related tools behind **subagents** (a research subagent owns search/fetch/summarize; a coordinator delegates to it), or behind **separate MCP servers** (a "GitHub server," a "database server"), exposing only the relevant subset to a given agent. Each agent then sees a small, coherent toolbox. This also improves prompt caching and keeps context lean (ties to D1 orchestration and D5 context management).

Prefer **consolidated, workflow-shaped tools** over many thin CRUD primitives. One `schedule_meeting(attendees, window)` that internally checks availability and books beats four low-level calendar tools the model must chain. Fewer, higher-level tools mean fewer round-trips, fewer error surfaces, and better selection.

Exam signal: A support or productivity agent is "unreliable" and "picks the wrong tool." Count the tools in the stem. If it's ~15+, the answer is **reduce/consolidate and split via subagents or MCP servers**, not "improve the prompt" or "add examples."

## tool_choice: Controlling Whether and Which Tool Runs

The `tool_choice` parameter controls tool invocation. `auto` (default when tools are present) lets the model decide. `any` forces it to call *some* tool (no plain-text answer). `tool` with a name forces a *specific* tool. There is also `none` to disable tool use for a turn. Use forced choices for deterministic flows: in **S6**, set `tool_choice` to your extraction tool so every response is a structured `tool_use` you can validate, rather than hoping the model emits JSON. Note that forcing a tool (`tool_choice` `any` or a named `tool`) is incompatible with extended thinking and returns an error — only `auto`/`none` are allowed when extended thinking is enabled, and they preserve the model's option to reason or answer directly.

Exam signal: "We need every response to be valid structured output for the pipeline." The lever is `tool_choice` set to the schema-bearing tool (plus schema validation + retry), not prompt-pleading for "always return JSON" (anti-pattern #3, prompt-based enforcement of a critical rule).

## MCP Architecture: Host, Client, Server

MCP standardizes how applications give models context and tools. Three roles: the **host** is the AI application (Claude Desktop, Claude Code, your Agent SDK app) that the user interacts with; it spawns one **client** per server, and each client maintains a 1:1 stateful connection to one **server**. The **server** is a separate program exposing capabilities (your filesystem server, GitHub server, internal API wrapper). This 1:host-to-many-clients, 1:client-to-server topology is the mental model to memorize. MCP is "USB-C for AI" — a uniform connector so any compliant client can talk to any compliant server.

## MCP Primitives: Tools, Resources, Prompts (+ Roots, Sampling, Elicitation)

**Server-exposed primitives** (what a server offers the host):
- **Tools** — model-controlled actions the model may invoke (functions with input schemas). This is the overlap with native API tool use.
- **Resources** — application/host-controlled context: file-like data identified by URI (documents, records, logs) the host can read and feed to the model. Read-only context, not actions.
- **Prompts** — user-controlled, reusable templates/workflows (e.g., a slash command or "summarize this PR" template) the user invokes intentionally.

**Client-exposed primitives** (what the host offers back to the server):
- **Roots** — filesystem/URI boundaries the host grants, scoping where a server may operate (least privilege).
- **Sampling** — the server can request an LLM completion *through the host* (the server gets model intelligence without holding its own API key; the host stays in control and can require approval).
- **Elicitation** — the server can request additional input from the *user* mid-operation (e.g., "which environment?").

Mnemonic for control: **tools = model-controlled, resources = app/host-controlled, prompts = user-controlled.** Mixing these up is a classic distractor (e.g., calling a read-only document feed a "tool," or claiming the model triggers prompts).

Exam signal: A stem describes exposing read-only reference docs to the agent. That's a **resource**, not a tool. A stem where a server needs the model to summarize without its own key → **sampling**. A server that must ask the user a clarifying question → **elicitation**.

## MCP Transports: stdio vs Streamable HTTP vs Legacy SSE

Two standard transports. **stdio** — the host launches the server as a local subprocess and talks over stdin/stdout. Best for local, single-user, low-latency integrations (local filesystem, a CLI tool); no network exposure, simplest auth (inherits the local user), but doesn't scale to remote/multi-user. **Streamable HTTP** — the server runs as an independent (often remote) service; the client POSTs JSON-RPC and the server can stream responses (including via SSE) over one endpoint. Best for remote, shared, or hosted servers; supports many clients and standard web auth (OAuth 2.1). The older **HTTP+SSE** two-endpoint design is **legacy/deprecated** in favor of streamable HTTP — recognize it but prefer streamable HTTP for new remote work.

Exam signal: "Local dev tool, one user, on the same machine" → stdio. "Remote, multi-tenant, needs auth and to scale" → streamable HTTP. If an answer proposes the legacy two-endpoint SSE design for a new remote server, it's the trap.

## Capability Negotiation and Auth (OAuth 2.1)

MCP connections begin with **capability negotiation**: during `initialize`, client and server exchange protocol version and advertise which features they support (does the server offer tools/resources/prompts? does the client support sampling/roots?). Neither side assumes capabilities; each declares them. This makes the protocol forward/backward compatible.

For **remote** servers, MCP authorization is built on **OAuth 2.1** (with PKCE). The MCP server acts as an OAuth resource server; tokens are obtained via standard authorization-server flows and presented per request. stdio servers typically don't need this (they inherit local credentials/env), which is part of why stdio is simpler but local-only.

## MCP Server Boundaries & Security: Least Privilege, Confused Deputy, Token Passthrough

MCP servers sit between the model and real systems, so **boundaries are security boundaries.** Apply **least privilege**: scope each server narrowly (use roots to bound filesystem access; grant the minimal API scopes), and don't expose one mega-server with broad credentials when several scoped servers would do.

**Confused-deputy** problem: a server holds powerful credentials and acts on behalf of a less-privileged caller; if it forwards requests without checking the *caller's* authorization, the model (or a malicious input) can trick the privileged server into doing something the caller shouldn't be allowed to do. Defense: the server validates the requester's authority for each action, doesn't blindly trust upstream tokens for downstream services, and enforces its own access checks.

**Token passthrough** is an explicit MCP anti-pattern: a server accepting a token issued for *it* and blindly forwarding that same token to a downstream API (or accepting tokens not minted for this server) breaks audience/scoping guarantees and audit trails. Each hop should use tokens scoped to that hop. Combine with input validation (the model's arguments are untrusted), and treat tool descriptions/resources from third-party servers as potential prompt-injection vectors.

Exam signal: A remote MCP server with one admin token serving many users, forwarding that token to internal APIs → flag confused-deputy + token passthrough; the fix is per-caller authorization, scoped/exchanged tokens, and least-privilege server boundaries — not "add a note to the prompt."

## MCP in Claude Code: .mcp.json and Scopes

Claude Code consumes MCP servers via configuration with three **scopes**: **local** (default; this project, only you — personal/experimental), **project** (shared with the team via a checked-in **`.mcp.json`** at the repo root — everyone who clones gets the same servers), and **user** (available to you across all your projects). Choose scope by who should see the server: team-shared integrations belong in `.mcp.json` (project scope); personal credentials/experiments stay local or user. You can add servers via `claude mcp add` (stdio, SSE, or HTTP) and reference env vars rather than hardcoding secrets. Servers can be approved/trusted before use to mitigate untrusted-server risk.

Exam signal (S4): "The whole team should get the internal docs MCP server automatically." Answer: **project scope via a committed `.mcp.json`.** "Just my machine, my token" → local/user scope. Distractors put secrets in the committed file or use the wrong scope.

## Decision: Build a Tool vs an MCP Server vs a Subagent

- **Tool** (native API tool / single function): you need one capability for *one* agent/app, tightly coupled, no reuse across apps. Cheapest. Add it to that agent's curated set.
- **MCP server**: the capability should be **reusable across multiple hosts/agents**, owned by a separate team, or wraps an external/internal system behind a stable, language-agnostic interface. Build a server when the integration is a shared product surface (GitHub, your data warehouse), or when you want host-independent auth/transport. Scope it narrowly (least privilege).
- **Subagent**: the work needs its *own* context window, its *own* curated toolset, and a focused role — and you want to keep the parent's context clean (D1/D5). Use a subagent to *contain* a large tool group (so the parent sees one delegation interface, not 15 tools) or to run an independent task (parallel research, fresh-context review — anti-pattern #9 fix).

Rule of thumb: **one capability for one agent → tool; reusable across agents/apps → MCP server; needs isolated context + its own tools → subagent.** These compose: a subagent can consume MCP-server tools.

Exam signal: "Three different internal apps all need to query the warehouse safely." That's an **MCP server** (reusable, scoped), not three duplicated tools and not a subagent.

## Top Traps (Domain 2)

1. Thin tool descriptions / loose `input_schema` → model mis-selects or mis-calls. Fix the schema and description, not the system prompt.
2. Generic error messages (#6) — strip diagnostics the model needs to recover.
3. Silently returning empty results as success (#7) — failure masked as data; set `is_error: true` with detail.
4. Too many tools on one agent (#8, the ~18-tool agent) — consolidate to ~4–7 and split via subagents/MCP servers.
5. Prompt-pleading to enforce structured output instead of `tool_choice` + schema validation (#3).
6. Confusing MCP primitive control: tools=model, resources=host, prompts=user. Don't call read-only context a "tool."
7. Legacy two-endpoint HTTP+SSE for a new remote server — use streamable HTTP.
8. Putting secrets in a committed `.mcp.json` / wrong MCP scope in Claude Code.
9. Confused-deputy + token passthrough on remote servers — enforce per-caller authz and scoped tokens; apply least privilege.
10. Overlapping tool names (`search`/`find`/`query` on the same data) causing selection oscillation.
