# Model Context Protocol (MCP) — Deep Reference for CCA-F

Study reference for the Claude Certified Architect – Foundations exam. Covers MCP architecture, primitives, transports, authorization, and Claude Code integration. Maps primarily to **D2 Tool Design & MCP Integration (18%)**, with strong ties to **S1 Customer Support Resolution Agent** and **S4 Developer Productivity**. Sources: [modelcontextprotocol.io](https://modelcontextprotocol.io) and [code.claude.com/docs/en/mcp](https://code.claude.com/docs/en/mcp) (formerly docs.anthropic.com/en/docs/claude-code/mcp).

## Why MCP Exists — "USB-C for AI Tools"

MCP is an open standard that defines a single, uniform way for AI applications to connect to external tools, data, and prompts. The canonical analogy is **"USB-C for AI tools"**: instead of writing a bespoke integration for every model-to-tool pairing (an M×N problem), you write each tool once as an MCP server and each AI app once as an MCP client (an M+N problem). Any compliant host can then talk to any compliant server.

MCP is built on **JSON-RPC 2.0** and is a **stateful, session-based** protocol. A session begins with an initialize handshake, persists capability state, and supports bidirectional requests and notifications. This statefulness matters: it is what enables server-initiated requests (sampling, elicitation) and live `list_changed` notifications.

**Exam signal (D2):** When a scenario asks "how should the agent integrate with the ticketing system / database / monitoring tool?", the architecturally correct answer is "expose it as an MCP server" — not "hardcode an API client into the agent" or "paste data into the prompt." MCP gives you reuse, isolation, and a standard auth/permission surface.

## Host / Client / Server Architecture and 1:1 Sessions

MCP uses a **client-host-server** architecture. Memorize the three roles and their exact responsibilities:

- **Host** — the AI application process (e.g., Claude Code, Claude Desktop, an Agent SDK app). The host *creates and manages multiple client instances*, controls connection permissions and lifecycle, enforces security policies and user consent, handles authorization decisions, coordinates the LLM and **sampling**, and aggregates context across clients. The host is the security boundary.
- **Client** — created by the host; each client maintains **one stateful session with exactly one server**. The client handles protocol negotiation and capability exchange, routes JSON-RPC messages bidirectionally, manages subscriptions/notifications, and maintains isolation between servers.
- **Server** — exposes resources, tools, and prompts via MCP primitives. Servers operate independently with focused responsibilities, can request sampling through the client, and may run as local processes or remote services.

The defining structural fact: **a host runs many clients, but each client has a strict 1:1 relationship with a single server.** One client, one server, one session.

**Security-critical design principle:** Servers **cannot read the whole conversation and cannot "see into" other servers.** Each server receives only the contextual information it needs; full conversation history stays with the host; cross-server interaction is mediated by the host. This isolation is the basis for several correct exam answers about least privilege and blast-radius containment.

**Exam signal (D2/S4):** A distractor will claim that connecting "one big MCP server with 18 tools" is simpler. The correct pattern is **composability**: multiple focused, single-responsibility servers, each easy to build and combine. This also dovetails with anti-pattern #8 (too many tools per agent) — split capabilities across server/subagent boundaries.

## Server Primitives — Tools, Resources, Prompts (Who Controls Each)

MCP defines three **server-side** primitives. The exam loves to test *who initiates/controls* each one — this is the single most testable MCP distinction.

- **Tools — model-controlled.** Executable functions the model decides to call (query a database, send an email, create a ticket). The application/host typically still gates execution with human-in-the-loop approval, but the *decision to invoke* originates from the model. Tools are the primitive behind agentic action.
- **Resources — application/data-controlled (app-driven).** Read-only contextual data identified by URIs (files, schemas, documents, records). Resources are surfaced and selected by the **application/user**, not autonomously invoked by the model. Think "data the app attaches as context," e.g. `file://`, `schema://`, `issue://` URIs. Resources can be subscribed to for updates when the server declares subscription support.
- **Prompts — user-controlled templates.** Predefined, parameterized prompt/workflow templates that a **user** explicitly invokes (e.g., a slash command). They standardize repeatable interactions.

A memory hook: **Tools = model picks, Resources = app/data provides, Prompts = user triggers.** If a question describes "the model autonomously calling X," X is a tool. If it describes "attaching a schema/document for context," that is a resource. If it describes "a reusable templated command a person runs," that is a prompt.

**Exam signal (D2/S6):** In S6 (Structured Data Extraction), the JSON schema you want Claude to extract against is naturally a **tool** (`tool_use` to enforce structure) — but a reference schema document the app supplies for grounding is a **resource**. Don't conflate the two.

## Client Primitives — Roots, Sampling, Elicitation

Three **client-side** capabilities let a server reach back into the host. These require the client to declare support during initialization.

- **Roots** — the client exposes filesystem boundaries (URIs, typically directories) that scope where a server may operate. In Claude Code, a stdio server can call `roots/list` to learn the directory Claude Code was launched from, and also reads the `CLAUDE_PROJECT_DIR` environment variable for the project root.
- **Sampling** — a server requests that the **client/host run an LLM completion** on its behalf. The host stays in control of model choice, cost, and consent — the server never gets direct model access or API keys. Sampling lets a server add intelligence (e.g., summarize, classify) while the host enforces policy.
- **Elicitation** — a server requests **structured input from the user** mid-task. Claude Code renders this automatically as either a **form-mode** dialog (server-defined fields) or a **URL-mode** flow (open a browser for auth/approval, then confirm in the CLI). No client config is required; an `Elicitation` hook can auto-respond.

**Exam signal (D2/S1):** Sampling and elicitation are the answer to "how does the server get more intelligence or more user input *without* breaking the host's security boundary or holding its own model credentials?" The host mediates; the server requests.

## Transports — stdio, Streamable HTTP, and Legacy HTTP+SSE

MCP messages (JSON-RPC) travel over a **transport**. Know the three and their trade-offs:

- **stdio (local)** — the host launches the server as a subprocess and communicates over stdin/stdout. Lowest latency, no network exposure, ideal for local tools needing direct system access (filesystem, git, local DB driver). Trade-off: runs only on the user's machine; not shareable as a hosted service; the host manages the process lifecycle (stdio servers are **not** auto-reconnected if they die).
- **Streamable HTTP (remote)** — the recommended transport for remote/cloud servers. A single HTTP endpoint handles requests and can stream responses; supports OAuth, scaling, and multi-user hosting. In Claude Code config the `type` field uses `http`, with `streamable-http` accepted as an alias (the spec's name), so vendor configs copy in cleanly.
- **Legacy HTTP+SSE (deprecated)** — the older two-channel Server-Sent Events transport (`--transport sse`). **Deprecated in favor of streamable HTTP**; use HTTP where available. Still seen in older server docs.

Rule of thumb: **local + direct system access → stdio; remote/cloud/multi-user + OAuth → streamable HTTP.** "SSE" appearing as the *recommended* choice is a stale/wrong answer.

**Exam signal (D2/S4):** "We want the whole team to share this integration via version control and authenticate with OAuth" → remote streamable **HTTP** server. "We need a server that reads local files and runs git directly" → **stdio**.

## Capability Negotiation and the initialize Handshake

Every session opens with **capability negotiation**. The client sends an `initialize` request declaring its capabilities (e.g., sampling, elicitation, roots, notification handling); the server responds with the capabilities it supports (e.g., tools, resources, prompts, resource subscriptions, `list_changed` notifications). Both parties must honor declared capabilities for the rest of the session, and features are negotiated **progressively** — the core protocol is minimal and extensible.

Consequences worth memorizing: a server can only emit resource-subscription notifications if it declared subscription support; tool invocation requires the tool capability; sampling requires the *client* to have declared sampling support. After init, the session runs three message loops: **client requests** (tools/resources, user- or model-initiated), **server requests** (sampling, forwarded by the client to the host's LLM), and **notifications** (resource updates, status changes, `list_changed`).

**Exam signal (D2):** "Why didn't the server's sampling request work?" — because the client/host never declared sampling capability during initialize. Capabilities are declared up front, not assumed.

## Authorization — OAuth 2.1 for Remote Servers

Remote MCP servers authenticate with **OAuth 2.1**. stdio (local) servers do not use OAuth — they inherit the user's local environment and credentials passed via env vars.

Claude Code's OAuth behavior, worth knowing concretely:

- A remote server is flagged as needing auth when it returns **401 Unauthorized** or **403 Forbidden**; you complete the flow with the **`/mcp`** command in-session (browser login). Tokens are stored securely (system keychain on macOS) and refreshed automatically.
- Discovery follows the standards chain: **RFC 9728 Protected Resource Metadata** at `/.well-known/oauth-protected-resource`, then **RFC 8414 authorization server metadata** at `/.well-known/oauth-authorization-server`. Servers using **Dynamic Client Registration** or a **Client ID Metadata Document (CIMD)** are auto-discovered.
- Scope control: set `oauth.scopes` (a space-separated string, RFC 6749 §3.3 format) to **pin a security-approved subset** of scopes even when the server advertises more. This is the supported, root-cause-correct way to enforce least privilege — not blocking the whole server.

**Exam signal (D2/S1):** Least-privilege auth = pin scopes / register a fixed callback port / use OAuth for remote servers. A distractor will suggest pasting a long-lived admin token as a static header for everything — that violates least privilege and confused-deputy safety.

## MCP in Claude Code — Adding Servers, `.mcp.json`, and Scopes

Add servers with `claude mcp add`. **All options come before the server name; `--` separates the name from the launch command.** Core forms:

```bash
# Remote streamable HTTP (recommended for cloud services)
claude mcp add --transport http notion https://mcp.notion.com/mcp

# Remote with a static bearer header
claude mcp add --transport http secure-api https://api.example.com/mcp \
  --header "Authorization: Bearer your-token"

# Local stdio server (note the -- before the command)
claude mcp add --transport stdio --env AIRTABLE_API_KEY=KEY airtable \
  -- npx -y airtable-mcp-server
```

Management: `claude mcp list`, `claude mcp get <name>`, `claude mcp remove <name>`, and the in-session `/mcp` panel (shows status, tool counts, and OAuth login). You can also `claude mcp add-json`, `claude mcp add-from-claude-desktop`, or run `claude mcp serve` to expose Claude Code *itself* as a stdio MCP server.

**Three installation scopes** — high-yield, memorize the storage location and sharing semantics:

- **local** (default) — current project only, private to you, stored in `~/.claude.json` under the project path. (Older versions called this `project`.) Use for personal/experimental servers and credentials you keep out of version control. Note: this is *not* the same file as `.claude/settings.local.json`.
- **project** — current project only, **shared with the team via `.mcp.json` at the project root** (checked into version control). Claude Code **prompts for approval** before using project-scoped servers (security gate); reset with `claude mcp reset-project-choices`.
- **user** — **all your projects**, private to you, stored in `~/.claude.json`. (Older versions called this `global`.) Use for personal utilities you want everywhere.

**Precedence** when the same server name (or, for plugins/connectors, the same endpoint) appears in multiple places: **local → project → user → plugin-provided → claude.ai connectors.** The highest-precedence definition is used **whole** — fields are not merged across scopes.

`.mcp.json` supports environment-variable expansion (`${VAR}` and `${VAR:-default}`) in `command`, `args`, `env`, `url`, and `headers` — this is how teams share one config while keeping secrets and machine-specific paths out of version control. A missing required variable with no default fails the parse.

**Exam signal (D3/S4):** "Share the same MCP tools with the whole team via git" → **project scope** (`--scope project`, `.mcp.json`). "Available across all my projects but private" → **user scope**. "Just me, just this project, with a secret token" → **local scope** (default).

## Using MCP Resources and Prompts in Claude Code

**Resources via `@` mentions.** Type `@` to autocomplete resources from all connected servers alongside files. Reference with `@server:protocol://resource/path`, e.g. `@github:issue://123`, `@docs:file://api/authentication`, `@postgres:schema://users`. Referenced resources are fetched and attached as context. This is how an app *pulls in data* (resource = app/data-controlled).

**Prompts via slash commands.** MCP prompts appear as `/mcp__servername__promptname` (spaces normalized to underscores). Run with space-separated args, e.g. `/mcp__github__pr_review 456` or `/mcp__jira__create_issue "Bug in login flow" high`. Prompt results are injected directly into the conversation (prompt = user-controlled).

**Tool Search (default on):** MCP tool definitions are deferred — only names load at session start, and Claude searches for tools on demand — so adding many servers barely costs context. Controlled by `ENABLE_TOOL_SEARCH` (`auto`/`auto:N` for threshold loading, `false` to disable). Per-server `alwaysLoad: true` exempts a small, always-needed tool set from deferral. This is the reliability-conscious way to scale MCP without blowing the context window (ties to **D5**).

## Remote vs Local Servers — Choosing Correctly

- **Local (stdio):** direct system access, lowest latency, runs on the user's machine, host-managed process, no OAuth (env-var credentials), no auto-reconnect. Best for filesystem/git/local-DB tooling and custom scripts.
- **Remote (streamable HTTP):** multi-user, shareable, OAuth-secured, auto-reconnects with exponential backoff (Claude Code: up to five attempts, 1s doubling). Best for SaaS integrations (Sentry, GitHub, Notion, Slack) and anything the team shares.

## Security Pitfalls — Confused Deputy, Token Passthrough, Over-Broad Exposure

MCP's flexibility creates specific, testable security failure modes. The architect must design these out at the root, never paper over them (consistent with "fix the root cause" guidance).

- **Confused-deputy problem** — a server (the deputy) holding broad credentials gets tricked into performing actions on behalf of a less-privileged caller. Mitigate with **least-privilege scopes** (`oauth.scopes` pinning), per-user consent enforced by the host, and not granting a server more authority than the task needs.
- **Token passthrough** — forwarding/reusing a token issued for one audience to a different downstream service. This breaks the audience/consent model and is an OAuth 2.1 anti-pattern. Each server should use tokens minted for *its* audience; don't blindly relay bearer tokens. Prefer per-server OAuth flows over one shared static admin header.
- **Over-broad tool exposure** — handing an agent a sprawling, unscoped toolset increases prompt-injection blast radius and confuses tool selection. Curate a **tight set (~4–7 tools)** per agent, split capabilities across focused servers/subagents, and remember the host's isolation guarantee (a server can't see other servers or the whole conversation) is your containment boundary.
- **Prompt injection via tool output** — content fetched by an MCP server (issues, web pages, files) can carry injected instructions. Verify/trust servers before connecting, keep destructive tools behind human approval, and never let model-controlled tools take irreversible action without a deterministic gate (ties to anti-pattern #3: enforce critical rules in code/hooks, not the prompt).

**Exam signal (D2/S1/S4):** When a scenario presents an over-privileged or over-broad MCP setup, the correct remediation is **narrow scopes + curated tools + host-enforced consent + per-audience tokens** — fixing the design — not "add a warning to the system prompt" or "trust the model to behave."

## Quick-Reference Cheat Sheet

- Roles: **Host** (many clients, security boundary, owns LLM/sampling) → **Client** (1:1 with one server) → **Server** (focused, isolated).
- Server primitives: **Tools = model-controlled**, **Resources = app/data-controlled**, **Prompts = user-controlled**.
- Client primitives: **Roots** (scope), **Sampling** (server asks host's LLM), **Elicitation** (server asks the user).
- Transports: **stdio** (local, no OAuth) / **streamable HTTP** (remote, OAuth, recommended) / **HTTP+SSE** (deprecated).
- Claude Code scopes: **local** (default, `~/.claude.json`, private) / **project** (`.mcp.json`, team via git, needs approval) / **user** (`~/.claude.json`, all projects, private). Precedence: local > project > user > plugin > claude.ai.
- Resources = `@server:proto://path` mentions; Prompts = `/mcp__server__prompt args` commands.
- Auth: OAuth 2.1 for remote; `/mcp` to log in; pin `oauth.scopes` for least privilege.
- Security: confused deputy, token passthrough, over-broad exposure → fix with scopes, per-audience tokens, curated tools, host-enforced consent.
