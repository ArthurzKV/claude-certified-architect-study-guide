# Current Claude API Errata & Version-Sensitive Notes (2026)

This sheet corrects a handful of **version-sensitive** details that drift fast on the Claude platform. It is authoritative over any older phrasing elsewhere in this corpus. Several community study guides (and parts of this corpus's base question bank) were written against older model behavior; the items below reflect **current** Claude models (Opus 4.6 / 4.7 / 4.8, Sonnet 4.5 / 4.6, Haiku 4.5).

The CCA-F exam is *Foundations* and mostly tests **architecture concepts** (validation-retry, structured output, escalation, stop_reason-based termination), not exact parameter names — but where a question turns on a concrete API behavior, prefer the current behavior below.

Sources: platform.claude.com/docs (structured outputs, adaptive thinking, effort, tool use, handling stop reasons, prompt caching, batch), docs.anthropic.com/en/docs/claude-code (settings), modelcontextprotocol.io (authorization spec).

---

## 1. Assistant-message prefill was removed on current models

What changed: sending a **prefilled final `assistant` turn** (e.g. seeding `{` to force JSON or skip preamble) now returns a **400 `invalid_request_error`** — "Prefilling assistant messages is not supported for this model" — on **Opus 4.6+ and Sonnet 4.5+** (and all newer models). This was a hard breaking change, not a graceful deprecation. Older models (Claude 3.5 Sonnet, Claude 3 Opus) still accept prefill, and **mid-conversation (non-final) assistant messages remain allowed**.

Current replacements for what prefill used to do:
- Skip preamble / force "JSON only" → a firm **system-prompt instruction** ("Respond with only the JSON object, no preamble").
- Guarantee JSON shape → **`output_config.format`** (JSON outputs) or **`strict: true`** tools (see §3).

Exam framing: if a question's "correct" answer is "prefill `{`", treat that as legacy. On current models the equivalent answer is a system-prompt instruction + structured outputs.

## 2. Adaptive thinking + `effort` replace fixed `budget_tokens`

What changed: current models use **adaptive thinking** — set `thinking: {type: "adaptive"}` and control reasoning depth with the **`effort`** parameter, which the model calibrates against query complexity. `effort` lives inside the **`output_config`** object in the request body (not inside `thinking`).

The older **`budget_tokens`** extended-thinking control is **legacy** (still functional on some 4.6-class models). For a hard cost ceiling on current models, lower `effort` or cap `max_tokens`. Thinking is **off by default**; when thinking is disabled the model is sensitive to the word "think" in prompts.

Unchanged concepts you still must know: thinking is for *depth not formatting* (pair with a schema/tool for the final answer); thinking is **incompatible with forced tool use** (`tool_choice` `any`/`tool` → error; use `auto`/`none`); and across tool-use turns **thinking blocks (with their signatures) must be passed back unmodified**.

## 3. Structured outputs are GA: `output_config.format` and `strict: true`

What changed: schema-guaranteed structured outputs are now **generally available — no beta header required** — on Opus 4.5/4.6/4.7/4.8, Sonnet 4.5/4.6, Haiku 4.5. Two complementary surfaces:
- **`output_config.format`** — constrains the **text response** to a JSON schema (JSON outputs).
- **`strict: true`** on a tool — guarantees the **tool input** conforms to its `input_schema` (grammar-constrained sampling). This is distinct from merely *forcing* a tool with `tool_choice`: forcing chooses the tool; `strict` guarantees its arguments' shape.

Hard limits of constrained decoding (high-yield, often mis-stated as "just less reliable"):
- Guarantees only **shape + `enum`/`const`**. It does **NOT** enforce `minimum`/`maximum`, `minLength`/`maxLength`, or `pattern` — the official SDKs **strip** these unsupported keywords, fold them into field **descriptions**, and **validate the response afterward**. So ranges/regex are *post-generation* validation, never decode-time guarantees.
- **No recursive schemas, no external `$ref`**; `additionalProperties` must be **`false`**.
- Supported subset includes `enum`, `const`, `anyOf`/`allOf`, internal `$ref`, `required`, string `format`.
- **Per-request complexity caps**: too many strict tools, too many optional parameters, or too many union-typed (`anyOf`) parameters → **400**. Remediation: reduce optional/union params, flatten, or split the request.

Therefore: even with strict outputs you still need a **post-generation validation + corrective-retry** layer for value ranges, cross-field rules, completeness (length/transport truncation), and semantics. Strict mode removes the *shape* problem, not the *semantics/completeness* problem.

## 4. Full `stop_reason` value set

The agentic loop terminates on `stop_reason`, never on natural-language "I'm done." Know the **complete** set, not just `end_turn`/`tool_use`/`max_tokens`/`stop_sequence`:
- `end_turn` — model finished naturally (stop the loop).
- `tool_use` — model wants a tool; execute it and continue the loop.
- `max_tokens` — output hit the cap (**truncated/incomplete**). If the last block is an **incomplete `tool_use`, do not execute partial args — retry with a higher `max_tokens`.**
- `stop_sequence` — a custom stop sequence was hit.
- `pause_turn` — a **server-tool sampling loop** (e.g. `web_search`/`web_fetch`, default ~10 internal iterations) paused mid-turn; **resume by re-sending the assistant response unmodified**.
- `refusal` — the model declined for safety; on Opus 4.7+ it carries `stop_details.category` (e.g. `cyber`/`bio`/`null`).
- `model_context_window_exceeded` — the context-window ceiling was hit before `max_tokens` (default-on for Sonnet 4.5+; otherwise behind a beta header). Treat as a context-budget signal (compact / trim, don't just bump `max_tokens`).

In **streaming**, `stop_reason` is populated on the `message_delta` event (it is `null` at `message_start`).

## 5. MCP remote-server auth = OAuth 2.1 + PKCE

The MCP authorization spec mandates **OAuth 2.1** (IETF draft) with **mandatory PKCE**; the MCP server acts as an **OAuth 2.1 resource server**. Spec-level MUSTs: the server **must reject tokens not minted for it** (audience binding), **token passthrough is prohibited**, and the client must validate the issuer (`iss`)/metadata — PKCE alone is not sufficient. (Some materials say "OAuth 2.0"; read it as **2.1 + PKCE**.) Remote MCP is **HTTPS-only** (no stdio).

## 6. Claude Code settings precedence: Local outranks Project

`settings.json` precedence, **highest → lowest**: **enterprise/managed → command-line arguments → `.claude/settings.local.json` (Local, gitignored) → `.claude/settings.json` (Project, committed) → `~/.claude/settings.json` (User)**. The gitignored **Local** file **outranks** the committed **Project** file, so a developer can locally override shared project settings. Permission rule arrays **merge** across scopes (they do not fully override), and on the merged set **`deny` > `ask` > `allow`**.

Related Claude Code currency notes:
- **Custom slash commands merged into Skills**: a `.claude/commands/deploy.md` and a `.claude/skills/<name>/SKILL.md` both create `/deploy`; existing command files still work but **skills are the recommended path and win on a name clash**.
- **PreToolUse** hooks use `hookSpecificOutput.permissionDecision` with values **`allow` / `deny` / `ask` / `defer`** (plus `permissionDecisionReason`); the generic `{"decision":"block"}` form is for `UserPromptSubmit` / `PostToolUse` / `Stop`. Exit-code 2 semantics are **per-event** (PostToolUse exit 2 only surfaces feedback since the tool already ran; UserPromptSubmit exit 2 blocks and erases the prompt; Stop/SubagentStop exit 2 prevents stopping).
- `CLAUDE.local.md` is still supported (loads alongside `CLAUDE.md`); the gitignored-import pattern is specifically recommended for sharing personal notes **across git worktrees**, not as a blanket replacement.

## 7. Batch API concrete constraints

The Message Batches API is **asynchronous**, ~**50% cheaper**, results within ~**24 hours** (24h expiration window), up to **100,000 requests / 256 MB** per batch, and requires **`max_tokens >= 1`** (so `max_tokens: 0` cache pre-warming is unsupported inside batches). Use it for non-latency-sensitive bulk work (CI multi-pass review of many files, dataset labeling, eval runs); use synchronous/streaming for interactive PR feedback.

---

### How to use this sheet
When a base-bank question or older community guide hinges on prefill, `budget_tokens`, "OAuth 2.0", a 4-value `stop_reason` set, or Project-over-Local settings, apply the corrected behavior above. For pure architecture/decision questions, the concept (validate-and-retry, structured output, programmatic termination, least-privilege auth, deterministic enforcement via hooks) is what the exam rewards — and that is unchanged.
