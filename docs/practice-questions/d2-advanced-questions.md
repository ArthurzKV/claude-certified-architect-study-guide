# Domain D2 — Advanced & Edge-Case Questions

These advanced and edge-case practice questions extend the core D2 bank (`d2-questions.md`) into higher-difficulty territory and close coverage gaps the core bank under-represents: **strict tool use (`strict: true`, grammar-constrained sampling, JSON Schema limitations)**, the **Tool Search Tool (`defer_loading`, `tool_search_tool_regex/bm25`, `tool_reference`)**, the **server-side MCP connector (`mcp_servers`, `mcp_toolset`, allowlist/denylist, `mcp-client-2025-11-20` beta)**, **`tool_choice` × extended-thinking incompatibility and assistant-prefill behavior**, **`input_examples`**, **parallel-tool control (`disable_parallel_tool_use`)**, **OAuth 2.1 / PKCE / token-audience binding precision**, **MCP elicitation/sampling/roots edge cases**, and **prompt-cache interactions with tool definitions and `tool_choice`**.

All items follow the mandatory MCQ format. Sources verified against platform.claude.com/docs (tool use, strict tool use, tool search tool, MCP connector, define tools), modelcontextprotocol.io (authorization, security best practices), and docs.anthropic.com/en/docs/claude-code. Anchored mainly to **S1**, **S4**, and **S6**, with some **S3/S5** cross-cuts.

## Section A — Strict Tool Use & Grammar-Constrained Sampling

### Q1. In S6 you need a hard guarantee that the model's tool inputs always match your JSON Schema (correct types, no missing required fields). Which mechanism provides this?

- **A.** Setting `strict: true` on the tool definition, which constrains token sampling to schema-valid outputs (grammar-constrained sampling)
- **B.** Raising `max_tokens` so the model has room to format JSON
- **C.** Adding "always output valid JSON" to the system prompt
- **D.** Lowering temperature to 0 so output becomes deterministic

**Answer:** `A`

**Explanation:** `strict: true` compiles your `input_schema` into a grammar and constrains sampling so tool inputs always conform (e.g., `passengers: 2`, never `"two"` or `"2"`), eliminating the validate-and-retry loop for type errors. `max_tokens` governs length, prompt-pleading is anti-pattern #3 (no guarantee), and temperature 0 reduces variance but does not enforce schema validity. Strict mode is the deterministic guarantee.

**Tags:** `D2, S6, Advanced, strict-tool-use`

### Q2. A teammate enables `strict: true` but the request fails or behaves oddly because the schema uses an unsupported JSON Schema feature. What is the correct response?

- **A.** Remove `strict: true` permanently; strict mode is unreliable
- **B.** Wrap the whole schema in a single freeform string property
- **C.** Conform the schema to the supported JSON Schema subset (the same limitations as structured outputs) — e.g., set `additionalProperties: false`, avoid unsupported keywords — rather than abandoning the guarantee
- **D.** Increase the retry cap to force eventual success

**Answer:** `C`

**Explanation:** Strict mode uses the structured-outputs grammar pipeline, which supports only a subset of JSON Schema; the fix is to bring the schema into that subset (commonly requiring `additionalProperties: false` and avoiding unsupported keywords), not to discard strict mode. Collapsing to a freeform string throws away validation, and cranking the retry cap is anti-pattern #2 (leaning on a backstop). Make the schema compatible, keep the guarantee.

**Tags:** `D2, S6, Advanced, strict-tool-use`

### Q3. Your S1 support tools handle PHI and you want strict mode for type safety. Which constraint must you respect?

- **A.** Strict mode cannot be used in any HIPAA context
- **B.** You must disable prompt caching to use strict mode with PHI
- **C.** PHI is automatically redacted from schemas, so no action is needed
- **D.** Do not place PHI in the schema itself (property names, `enum`/`const` values, `pattern` regexes); strict mode caches compiled schemas separately, so PHI belongs only in message content

**Answer:** `D`

**Explanation:** Strict tool use is HIPAA-eligible, but compiled schemas are cached separately (up to ~24h) and do not get the same PHI protections as prompts/responses, so PHI must never appear in `input_schema` property names, enum/const values, or patterns — only in message content. It is not categorically barred from HIPAA, schemas are not auto-redacted, and disabling caching is not the requirement. Keep PHI out of the schema.

**Tags:** `D2, S1, Advanced, strict-tool-use`

### Q4. What does `strict: true` guarantee beyond the input shape in a tool_use response?

- **A.** That the tool `name` in the response is always a valid tool (from your provided tools or a server tool), in addition to the `input` matching the schema
- **B.** That the tool will never return an error
- **C.** That the model will always call exactly one tool
- **D.** That the model reasons before calling the tool

**Answer:** `A`

**Explanation:** Strict mode guarantees both that the `input` conforms to your `input_schema` and that the `name` is always a valid tool name. It says nothing about runtime tool errors, does not force a call count (that is `tool_choice`), and does not control reasoning. Strict constrains the structure of the call, not execution outcomes.

**Tags:** `D2, S6, Intermediate, strict-tool-use`

### Q5. For S6, how can you guarantee BOTH that a tool is called every turn AND that its arguments match the schema exactly?

- **A.** `tool_choice: {"type": "auto"}` alone
- **B.** A system-prompt instruction to "always call the tool with valid JSON"
- **C.** `strict: true` alone, which also forces a call
- **D.** Combine `tool_choice: {"type": "any"}` (or a forced specific tool) with `strict: true` on the tool definitions

**Answer:** `D`

**Explanation:** `tool_choice` controls *whether/which* tool is called; `strict: true` controls *whether the inputs conform*. Combining a forced choice (`any` or a named tool) with strict mode gives both guarantees. `auto` may answer in prose, `strict` alone does not force a call, and prompt-pleading is anti-pattern #3. These are orthogonal levers you compose.

**Tags:** `D2, S6, Advanced, strict-tool-use`

## Section B — Tool Search Tool, defer_loading & Large Tool Catalogs

### Q6. An S4 agent connects to GitHub, Slack, Sentry, Grafana, and Splunk MCP servers, and tool definitions alone consume tens of thousands of context tokens before any work begins. Which native capability best addresses this?

- **A.** Enable the Tool Search Tool and mark non-core tools `defer_loading: true`, so only the 3–5 tools Claude needs for a given request are loaded on demand
- **B.** Drop servers until the context fits
- **C.** Force `tool_choice` to one tool to reduce token use
- **D.** Lower `max_tokens` to leave room for tool definitions

**Answer:** `A`

**Explanation:** The Tool Search Tool keeps deferred tool definitions out of the upfront context and surfaces only the few relevant tools per request, typically cutting tool-definition tokens by a large margin while preserving selection accuracy across large catalogs. Dropping servers loses capability (a workaround, not a root fix), forcing one tool breaks the multi-tool workflow, and `max_tokens` is unrelated to input bloat. Use just-in-time tool discovery.

**Tags:** `D2, S4, Advanced, tool-search`

### Q7. When using the Tool Search Tool, which configuration is INVALID and will return a 400 error?

- **A.** The tool search tool is non-deferred and several other tools have `defer_loading: true`
- **B.** ALL tools (including the tool search tool itself) have `defer_loading: true`
- **C.** A mix of deferred and non-deferred tools with clear descriptions
- **D.** Keeping the 3–5 most-used tools non-deferred and deferring the rest

**Answer:** `B`

**Explanation:** At least one tool must be non-deferred, and the tool search tool itself must never be deferred — deferring everything yields a 400 ("All tools have defer_loading set"). The other options are valid, recommended patterns. The search tool must stay loaded so Claude can discover the deferred ones.

**Tags:** `D2, S4, Advanced, tool-search`

### Q8. What is the key difference between the regex and BM25 variants of the Tool Search Tool?

- **A.** Regex searches descriptions only; BM25 searches names only
- **B.** Regex is deprecated in favor of BM25
- **C.** BM25 cannot be used with MCP servers
- **D.** The regex variant has Claude construct Python `re.search`-style patterns; the BM25 variant has Claude issue natural-language queries — both search names, descriptions, and argument names/descriptions

**Answer:** `D`

**Explanation:** `tool_search_tool_regex_*` uses Python regex patterns (e.g., `(?i)slack`, max ~200 chars), while `tool_search_tool_bm25_*` uses natural-language queries; both search across tool names, descriptions, and argument names/descriptions. Neither is name-only or description-only, neither is deprecated, and both work with MCP-backed tool sets. Pick the query style that fits how you reason about your catalog.

**Tags:** `D2, S4, Advanced, tool-search`

### Q9. How does `defer_loading` interact with prompt caching of tool definitions?

- **A.** Deferred tools are excluded from the cached system-prompt prefix; when discovered, a `tool_reference` is appended inline and expanded later in the conversation, so the cached prefix stays intact
- **B.** It breaks the cache because deferred tools change the system-prompt prefix
- **C.** It forces a full cache rebuild on every search
- **D.** Caching is incompatible with the Tool Search Tool

**Answer:** `A`

**Explanation:** Deferred tool definitions are not part of the prompt-prefix, and discovered tools are injected as inline `tool_reference` blocks downstream, so the cacheable prefix is untouched and caching is preserved. It does not break or rebuild the cache, and the two features are designed to compose. Deferral preserves prompt caching by keeping the prefix stable.

**Tags:** `D2, S4, Advanced, tool-search`

### Q10. Claude isn't discovering an expected tool via the regex Tool Search Tool. What is the most likely root cause and fix?

- **A.** The tool's handler is throwing; set `is_error: true`
- **B.** `max_tokens` is too low
- **C.** The tool name, description, or argument names/descriptions don't match the (case-sensitive) pattern; improve the tool's discoverability keywords and/or use a case-insensitive pattern like `(?i)...`
- **D.** The model needs a higher temperature to search creatively

**Answer:** `C`

**Explanation:** Regex search is case-sensitive by default and matches against names, descriptions, and argument fields, so non-matching or keyword-poor metadata is the usual culprit; adding semantic keywords and/or using `(?i)` fixes it. It is not a handler error, a token limit, or a temperature issue. Make tools discoverable by writing keyword-rich, consistently namespaced metadata.

**Tags:** `D2, S4, Advanced, tool-search`

### Q11. Which design choice most improves Tool Search discoverability across a 200+ tool MCP estate in S4?

- **A.** Single-word generic names like `get`, `search`, `run`
- **B.** Removing argument descriptions to shrink the catalog
- **C.** Deferring the tool search tool itself to save tokens
- **D.** Consistent service/resource namespacing in tool names (e.g., `github_`, `slack_`, `jira_`) plus keyword-rich descriptions

**Answer:** `D`

**Explanation:** Consistent prefixes and keyword-rich descriptions let search queries naturally surface the right tool group and keep selection unambiguous as the catalog grows. Generic names cause overlap/oscillation (a D2 trap), the search tool must never be deferred, and stripping argument descriptions removes searchable signal. Namespacing + rich metadata drives discoverability.

**Tags:** `D2, S4, Advanced, tool-search`

## Section C — Server-Side MCP Connector (Messages API)

### Q12. Your S1 backend uses the Claude Messages API directly and must call a remote, OAuth-protected MCP server WITHOUT building a separate MCP client. Which feature fits?

- **A.** The MCP connector: pass the server in the `mcp_servers` array with an `authorization_token`, reference it via an `mcp_toolset` in `tools`, and send the MCP-client beta header
- **B.** Claude Code `.mcp.json` project scope
- **C.** The embeddings API
- **D.** A stdio transport configured in the request body

**Answer:** `A`

**Explanation:** The MCP connector lets the Messages API connect to remote MCP servers directly via the `mcp_servers` array (URL + `authorization_token`) plus an `mcp_toolset` entry, gated by the MCP-client beta header — no separate client needed. `.mcp.json` is Claude Code's client config (not the raw API), embeddings are unrelated, and the connector does not accept stdio servers. Use the MCP connector for direct-from-API remote MCP.

**Tags:** `D2, S1, Advanced, mcp-connector`

### Q13. Which MCP servers can the Messages API MCP connector talk to directly?

- **A.** Local stdio servers launched as subprocesses
- **B.** Only publicly reachable HTTPS servers (Streamable HTTP or SSE) — local stdio servers cannot be connected directly
- **C.** Any server regardless of transport
- **D.** Only servers running on Anthropic's infrastructure

**Answer:** `B`

**Explanation:** The connector requires the server be publicly exposed over HTTPS (Streamable HTTP or SSE); local stdio servers cannot be reached this way — for those you run your own client/SDK helpers. It is not transport-agnostic here and is not limited to Anthropic-hosted servers. Remote HTTP only for the connector.

**Tags:** `D2, S1, Advanced, mcp-connector`

### Q14. Through the current MCP connector, which MCP capabilities are exposed to the model?

- **A.** Only tool calls — resources and prompts are not currently supported via the connector
- **B.** Tools, resources, and prompts equally
- **C.** Only resources
- **D.** Sampling and elicitation

**Answer:** `A`

**Explanation:** The connector currently supports only MCP tool calls; resources and prompts are not exposed through it (for those, manage your own MCP client or use SDK helpers). It is not full-spectrum, not resource-only, and not a sampling/elicitation channel. Connector = remote MCP tools via the Messages API.

**Tags:** `D2, S4, Advanced, mcp-connector`

### Q15. You want the MCP connector to expose ONLY two specific tools from a connected server and hide the rest. What is the correct pattern?

- **A.** Edit the remote server's code to delete the other tools
- **B.** There is no way to restrict tools; all server tools are always exposed
- **C.** Put the two tool names in the tool description
- **D.** In the `mcp_toolset`, set `default_config.enabled: false` and enable just those two tools by name in `configs` (allowlist pattern)

**Answer:** `D`

**Explanation:** The MCP toolset supports allowlisting by setting `default_config.enabled: false` and turning on only the named tools in `configs` (and denylisting by disabling specific tools while others default on). You don't have to modify the remote server, descriptions don't gate exposure, and tool exposure is configurable per request. Allowlist via toolset config.

**Tags:** `D2, S4, Advanced, mcp-connector`

### Q16. For a read-only S4 assistant backed by a write-capable MCP server via the connector, what is the recommended safety configuration?

- **A.** Trust the model to avoid destructive tools via a prompt instruction
- **B.** Lower temperature so the model is less likely to call them
- **C.** Denylist the destructive/write tools in the `mcp_toolset` `configs` (set `enabled: false`) so they are never exposed, optionally requiring human confirmation for any state changes
- **D.** Set `is_error: true` on the write tools

**Answer:** `C`

**Explanation:** Disabling write/destructive tools at the toolset config layer is a deterministic control that removes them from the model's options entirely, far safer than prompt-pleading (anti-pattern #3) or hoping low temperature suppresses a call. `is_error` is a tool-result flag, not an exposure control. Enforce read-only posture by configuration, not persuasion.

**Tags:** `D2, S4, Advanced, mcp-connector`

### Q17. With the MCP connector, who is responsible for obtaining and refreshing the OAuth access token passed as `authorization_token`?

- **A.** Anthropic mints and refreshes it automatically
- **B.** The token is unnecessary because the connector bypasses auth
- **C.** The model requests it via elicitation at runtime
- **D.** You (the API consumer) complete the OAuth flow out-of-band, obtain the access token, pass it in the server definition, and refresh it as needed

**Answer:** `D`

**Explanation:** The connector supports passing an `authorization_token`, but the API consumer is expected to handle the OAuth flow and token refresh; Anthropic does not perform the flow for you. The model does not fetch tokens via elicitation here, and auth is not bypassed. You own token acquisition and lifecycle.

**Tags:** `D2, S1, Advanced, mcp-connector`

## Section D — tool_choice Edge Cases, Extended Thinking & Caching

### Q18. In S6 you enable extended thinking AND set `tool_choice: {"type": "tool", "name": "extract_fields"}`. What happens?

- **A.** It errors: forced tool choice (`any` or a specific `tool`) is NOT supported with extended thinking; only `auto` and `none` are compatible
- **B.** The model reasons, then calls the forced tool — fully supported
- **C.** Thinking is silently disabled and the tool is called
- **D.** The tool is ignored and the model answers in prose

**Answer:** `A`

**Explanation:** Per the tool-use docs, extended thinking is incompatible with `tool_choice: {"type": "any"}` and `{"type": "tool", ...}`, which return an error; only `auto` (default) and `none` work with thinking. It does not silently downgrade or ignore the tool. If you need pre-call reasoning, use `auto` and steer with prompting/`input` instructions.

**Tags:** `D2, S6, Advanced, tool-choice`

### Q19. With `tool_choice: {"type": "any"}` or a forced specific tool, what behavior should your S1 client expect in the assistant turn?

- **A.** Claude always emits a natural-language explanation before the tool_use block
- **B.** The API prefills the assistant turn to force a tool, so Claude will NOT emit preceding natural-language text — even if asked to
- **C.** Claude emits the tool_use only after a separate thinking block
- **D.** Claude alternates between text and tool calls unpredictably

**Answer:** `B`

**Explanation:** When `tool_choice` is `any` or `tool`, the API prefills the assistant message to force tool use, so there is no leading prose/explanation before the `tool_use` block. If you need conversational text alongside a specific tool, use `auto` and instruct the model in a user message. Forcing a tool suppresses the preamble.

**Tags:** `D2, S1, Advanced, tool-choice`

### Q20. You rely on prompt caching for S5's high-volume review pipeline. How does changing `tool_choice` between requests affect the cache?

- **A.** It has no effect on any cached content
- **B.** Caching is disabled whenever `tool_choice` is set
- **C.** It invalidates the entire cache including tool definitions
- **D.** Changing `tool_choice` invalidates cached *message* blocks (they must be reprocessed), though tool definitions and system prompts remain cached

**Answer:** `D`

**Explanation:** Per the docs, altering `tool_choice` invalidates cached message blocks (message content is reprocessed) while tool definitions and system prompts stay cached. It is neither a no-op nor a full cache wipe, and caching is not disabled by setting `tool_choice`. Keep `tool_choice` stable across cached requests to maximize hits.

**Tags:** `D2, S5, Advanced, prompt-caching`

### Q21. In S4, you want Claude to execute several INDEPENDENT tool calls (e.g., read five files) efficiently, but for one critical step you must force strictly sequential, one-at-a-time calls. Which lever controls this?

- **A.** `disable_parallel_tool_use` within the `tool_choice` object, which prevents the model from emitting multiple tool_use blocks in a single turn
- **B.** `max_tokens`
- **C.** Switching transports
- **D.** Lowering temperature

**Answer:** `A`

**Explanation:** Claude can emit multiple independent `tool_use` blocks per turn by default; setting `disable_parallel_tool_use: true` in the `tool_choice` object forces at most one tool call per turn for steps that must be sequential. Token limits, transport, and temperature do not control parallelism. Use `disable_parallel_tool_use` to serialize when ordering matters.

**Tags:** `D2, S4, Advanced, parallel-tools`

### Q22. Some Claude models or modes do not support forced tool use at all (forced choice returns a 400). What is the robust architectural guidance?

- **A.** Always assume forced tool use is available everywhere
- **B.** Hard-fail the pipeline if forced choice is unsupported
- **C.** Treat forced tool use as model/mode-dependent: when unavailable (or when using extended thinking), fall back to `tool_choice: auto`/`none` plus prompting and validate outputs in code
- **D.** Encode the forcing rule in the tool description

**Answer:** `C`

**Explanation:** Forced tool use availability varies by model and is incompatible with extended thinking, so resilient designs detect that and fall back to `auto`/`none` with prompting and code-side validation rather than assuming universal support. Hard-failing is brittle, and putting "you must call this" in a description is anti-pattern #3. Design for capability variation and validate deterministically.

**Tags:** `D2, S6, Advanced, tool-choice`

## Section E — input_examples & Tool-Definition Nuances

### Q23. Your S6 extraction tool has a complex, nested, format-sensitive `input_schema` and the model occasionally structures arguments incorrectly. Besides a rich description, what native field helps?

- **A.** Add prose examples inside the system prompt only
- **B.** Set `is_error: true` preemptively
- **C.** Add a second mega-tool that reformats the output
- **D.** Add `input_examples`: an array of schema-valid example input objects on the tool definition, shown alongside the schema to demonstrate well-formed calls

**Answer:** `D`

**Explanation:** The optional `input_examples` field provides concrete, schema-validated example inputs that help Claude structure complex/nested/format-sensitive arguments correctly. System-prompt prose is weaker and unstructured, a reformatting mega-tool grows count (anti-pattern #8), and `is_error` is a result flag. Use `input_examples` for hard-to-format inputs (note each example must validate against the schema, or you get a 400).

**Tags:** `D2, S6, Advanced, tool-schema-design`

### Q24. Which statement about `input_examples` is correct?

- **A.** They are supported on user-defined and Anthropic-schema client tools but NOT on server tools, each example must validate against the `input_schema`, and they add prompt tokens
- **B.** They work on server tools (web_search, code_execution) too
- **C.** They replace the need for an `input_schema`
- **D.** They are free in token cost

**Answer:** `A`

**Explanation:** `input_examples` apply to client tools (user-defined and Anthropic-schema), are rejected (400) if an example violates the schema, and add to prompt tokens; they are not available for server tools and do not replace the schema. They are a supplement to a precise schema, not a substitute.

**Tags:** `D2, S6, Advanced, tool-schema-design`

### Q25. Anthropic's tool-design guidance favors which structure for a set of related operations like `create_pr`, `review_pr`, `merge_pr`?

- **A.** Three separate thin tools, always
- **B.** Consolidating related operations into fewer, higher-level tools (e.g., one tool with an `action` parameter) to reduce selection ambiguity, while keeping each tool's responsibility coherent
- **C.** One mega-tool `do_everything(action, payload)` spanning unrelated domains
- **D.** A tool per database column touched

**Answer:** `B`

**Explanation:** Anthropic recommends consolidating closely related operations into fewer capable tools (e.g., an `action` enum) to cut selection ambiguity, which complements right-sizing to ~4–7 tools. This differs from a cross-domain `do_everything` mega-tool (opaque, loses precision) and from per-column tools (anti-pattern #8 explosion). Consolidate *related* operations; don't merge *unrelated* domains.

**Tags:** `D2, S4, Intermediate, tool-count`

### Q26. A tool `name` is rejected by the API. Which constraint was likely violated?

- **A.** Names must be at least 20 characters
- **B.** Names may not contain underscores
- **C.** Names must be globally unique across all Anthropic accounts
- **D.** Names must match `^[a-zA-Z0-9_-]{1,64}$` (letters, digits, underscore, hyphen; 1–64 chars)

**Answer:** `D`

**Explanation:** Tool names must satisfy `^[a-zA-Z0-9_-]{1,64}$`; spaces, slashes, or names over 64 chars are invalid. There is no 20-char minimum, no global-uniqueness rule, and underscores are allowed (and idiomatic for namespacing). Keep names within the allowed character set and length.

**Tags:** `D2, S4, Intermediate, tool-schema-design`

## Section F — MCP Auth, Token Audience & Security Precision

### Q27. Precisely, what OAuth foundation does MCP authorization for remote (HTTP) servers build on, and what does it require of clients?

- **A.** OAuth 2.1 (with mandatory PKCE), where the MCP server acts as an OAuth 2.1 resource server; stdio servers instead read credentials from the environment
- **B.** OAuth 1.0a with HMAC signatures
- **C.** Plain API keys in the URL query string
- **D.** SAML assertions only

**Answer:** `A`

**Explanation:** MCP's HTTP authorization is based on OAuth 2.1 with mandatory PKCE, treating the MCP server as a resource server; stdio transports should NOT use this flow and instead retrieve credentials from the environment. It is not OAuth 1.0a, not URL-embedded keys (tokens must not be in the query string), and not SAML. Know OAuth 2.1 + PKCE for remote, env credentials for stdio.

**Tags:** `D2, S1, Advanced, mcp-auth`

### Q28. Per the MCP spec, what MUST an MCP server do with access tokens it receives, and what is explicitly forbidden?

- **A.** Accept any valid-looking token and forward it to downstream APIs to simplify the chain
- **B.** Store tokens in tool descriptions for reuse
- **C.** Validate that tokens were issued specifically for it (audience binding) and reject tokens not minted for it; token passthrough (forwarding the received token unchanged to downstream services) is explicitly forbidden
- **D.** Trust the upstream caller's token without validation

**Answer:** `C`

**Explanation:** The spec requires audience validation — servers must reject tokens not intended for them — and explicitly forbids token passthrough (forwarding the client's token to upstream APIs), because that breaks audience guarantees and enables the confused-deputy problem. Forwarding arbitrary tokens, embedding them in descriptions, or trusting upstream blindly all violate the rules. When calling upstream, the server must use a *separate* token it obtains as a client.

**Tags:** `D2, S1, Advanced, mcp-security`

### Q29. Why does PKCE alone NOT fully protect the MCP authorization-code flow, per the spec's threat model?

- **A.** PKCE is optional and rarely implemented
- **B.** PKCE only applies to stdio servers
- **C.** PKCE encrypts tokens, which is unnecessary
- **D.** PKCE doesn't stop a malicious authorization server from intercepting the flow; clients must ALSO validate the authorization response against the recorded issuer (`iss`) they bound before redirecting

**Answer:** `D`

**Explanation:** PKCE prevents code interception/injection by an attacker who lacks the verifier, but if a malicious authorization server is in the loop the client may transmit the verifier to it; the spec therefore also requires binding/validating the response to the recorded `iss` (issuer). PKCE is mandatory (not optional), doesn't "encrypt tokens," and applies to HTTP, not stdio. Combine PKCE with issuer validation.

**Tags:** `D2, S1, Advanced, mcp-auth`

### Q30. A remote MCP server in S1 uses one shared admin token for all users and forwards it to internal APIs — which two coupled problems is this, and what is the correct fix?

- **A.** Confused-deputy + token passthrough; fix with per-caller authorization checks, audience-bound/exchanged tokens scoped to each hop, and least-privilege server boundaries
- **B.** Rate limiting and caching; raise limits and cache more
- **C.** Generic errors and silent suppression; add `is_error`
- **D.** Tool-count overload; reduce to ~4–7 tools

**Answer:** `A`

**Explanation:** A privileged server acting for less-privileged callers without checking their authority is the confused-deputy problem, compounded by token passthrough (forwarding the same token downstream); the fix is per-caller authz, tokens scoped/exchanged per hop with audience binding, and least-privilege boundaries. It is not a rate/cache, error-semantics, or tool-count issue. Enforce authority and scope at every hop.

**Tags:** `D2, S1, Advanced, mcp-security`

### Q31. A client fetches authorization-server metadata where the document's `issuer` differs from the issuer in the well-known URL it requested. What must the client do?

- **A.** Proceed; minor issuer mismatches are normal
- **B.** Reject the metadata and not use it — the `issuer` in the document MUST match the expected issuer, or it may be an attacker-supplied document
- **C.** Use it but downgrade to OAuth 2.0
- **D.** Cache it for later and retry

**Answer:** `B`

**Explanation:** The spec requires the `issuer` value in the metadata to be identical to the issuer used to build the well-known URL; on mismatch the client MUST reject the document (e.g., an attacker host returning someone else's issuer). Proceeding, downgrading, or caching the bad document all undermine the trust chain. Validate issuer identity strictly.

**Tags:** `D2, S1, Advanced, mcp-auth`

## Section G — MCP Primitives Edge Cases (Sampling, Elicitation, Roots, Resources)

### Q32. In S3, an MCP server needs the model to summarize a passage but must NOT hold its own API key or pick the model unilaterally. Which primitive applies, and what control does the host keep?

- **A.** Sampling; the server requests an LLM completion *through the host*, which stays in control (it can require approval and governs model/keys) so the server never needs its own key
- **B.** Elicitation; the host asks the user
- **C.** Roots; the host grants filesystem scope
- **D.** A Resource; the host attaches context

**Answer:** `A`

**Explanation:** Sampling lets a server request a model completion via the host, so the host retains control over approval, model selection, and credentials, and the server gets intelligence without its own key. Elicitation is for asking the *user* a question, roots bound filesystem scope, and resources are read-only context. Sampling = host-mediated LLM access for the server.

**Tags:** `D2, S3, Advanced, mcp-primitives`

### Q33. Midway through an operation, an MCP server must ask the human "which environment: staging or prod?" Which primitive is designed for this?

- **A.** Sampling
- **B.** A Tool the model calls
- **C.** Elicitation — the server requests additional input from the user mid-operation
- **D.** A Prompt template

**Answer:** `C`

**Explanation:** Elicitation lets a server request more input from the *user* during an operation (e.g., choosing an environment). Sampling is for getting an LLM completion, a Tool is a model-invoked action, and a Prompt is a user-selected template. Match "ask the user mid-flow" to elicitation.

**Tags:** `D2, S1, Advanced, mcp-primitives`

### Q34. An architect calls a read-only document feed exposed by an MCP server a "tool" and assumes the model triggers it. Why is this wrong on the exam rubric?

- **A.** It's correct; resources and tools are interchangeable
- **B.** The model controls all three primitives equally
- **C.** Resources are write actions, so it should be a Prompt
- **D.** Read-only context is a Resource (app/host-controlled), not a model-invoked Tool; conflating them mis-assigns who controls invocation (model vs host) and is a classic distractor

**Answer:** `D`

**Explanation:** The control mnemonic is tools = model-controlled, resources = app/host-controlled, prompts = user-controlled; a read-only feed is a Resource the host surfaces, not a model-triggered Tool. They are not interchangeable, resources are not writes, and the model does not control resources or prompts. Mapping the wrong control owner is the trap.

**Tags:** `D2, S4, Advanced, mcp-primitives`

### Q35. What is the purpose of MCP "roots," and which side exposes them?

- **A.** Client/host-exposed filesystem/URI boundaries that scope where a server may operate, enforcing least privilege
- **B.** Server-exposed actions the model calls
- **C.** A list of root CA certificates for TLS
- **D.** The top-level tools a server advertises

**Answer:** `A`

**Explanation:** Roots are a client-exposed primitive: the host grants filesystem/URI boundaries that bound a server's operating scope (least privilege). They are not server actions, not TLS roots, and not a tool list. Roots constrain server reach; the host offers them to the server.

**Tags:** `D2, S4, Advanced, mcp-primitives`

## Section H — Capability Negotiation, Transports & Cross-Cutting

### Q36. During MCP `initialize`, the client advertises sampling support but the connected server never declared a tools capability. What follows from capability negotiation?

- **A.** The client may still call tools; negotiation is advisory
- **B.** The server is forced to expose tools anyway
- **C.** Neither side assumes undeclared capabilities — if the server didn't advertise tools, the client must not expect tool calls from it; both sides only rely on capabilities exchanged at initialize
- **D.** Sampling is disabled because tools are absent

**Answer:** `C`

**Explanation:** Capability negotiation means each side declares what it supports at `initialize` and neither assumes undeclared features; if the server didn't advertise tools, the client shouldn't expect them. Negotiation is not merely advisory, the server isn't forced to add capabilities, and sampling support is independent of tools. Rely only on negotiated capabilities.

**Tags:** `D2, S4, Advanced, mcp-architecture`

### Q37. You are standing up a NEW remote, multi-tenant MCP server. An answer proposes the legacy two-endpoint HTTP+SSE design. Why reject it?

- **A.** SSE cannot carry JSON-RPC
- **B.** The legacy two-endpoint HTTP+SSE transport is deprecated in favor of the single-endpoint Streamable HTTP transport (which can still stream via SSE); choose Streamable HTTP for new remote work
- **C.** Remote servers must use stdio
- **D.** HTTP transports cannot authenticate

**Answer:** `B`

**Explanation:** The older two-endpoint HTTP+SSE design is legacy/deprecated; new remote servers should use Streamable HTTP (one endpoint, can stream over SSE, supports OAuth 2.1). SSE can carry JSON-RPC, remote servers don't use stdio, and HTTP transports do support auth. Recognize the legacy pattern as the trap and pick Streamable HTTP.

**Tags:** `D2, S4, Advanced, mcp-transports`

### Q38. An MCP server is "transport-agnostic" at the capability layer. What practical migration does this enable, and what does it NOT change?

- **A.** The same JSON-RPC capabilities (tools/resources/prompts) behave identically over stdio or HTTP, so you can migrate a server local→remote without changing its capability contract — but transport choice does not auto-encrypt or alter which primitives exist
- **B.** Moving stdio→HTTP automatically encrypts all data
- **C.** Switching transports changes which primitives are available
- **D.** HTTP servers cannot expose tools

**Answer:** `A`

**Explanation:** Because capabilities are independent of transport, a server's tools/resources/prompts stay the same across stdio and HTTP, easing local-to-remote migration; however, transport does not auto-encrypt data nor change the primitive set. The other options misstate transport effects. Capability contract is stable; security (TLS/auth) is a separate, deliberate concern.

**Tags:** `D2, S4, Advanced, mcp-transports`

### Q39. In S5's CI pipeline you batch hundreds of structured-extraction jobs and want each to call a remote MCP tool. Which is true?

- **A.** MCP connector and Tool Search cannot be used in the Batches API
- **B.** Batch mode disables tool use entirely
- **C.** You can include `mcp_servers` (connector) and the Tool Search Tool in Messages Batches API requests, priced the same as regular Messages requests
- **D.** Batch requests must use stdio MCP servers

**Answer:** `C`

**Explanation:** Both the MCP connector (`mcp_servers`) and the Tool Search Tool are usable in the Messages Batches API, priced identically to standard Messages requests. Batch mode does not disable tools, and the connector still requires remote HTTP servers (not stdio). Batch composes with remote MCP tools and tool search.

**Tags:** `D2, S5, Advanced, mcp-connector`

### Q40. For an S4 task that genuinely needs many capabilities, which is the right strategy rather than the anti-pattern?

- **A.** Put all 20 tools on one agent and rely on a longer system prompt to disambiguate
- **B.** Rename tools with numeric prefixes to impose an order
- **C.** Force one tool per turn so only one is ever considered
- **D.** Curate ~4–7 core tools on the primary agent and handle the rest via subagents, scoped MCP-server boundaries, and/or the Tool Search Tool with `defer_loading` — partitioning by coherent responsibility

**Answer:** `D`

**Explanation:** The correct pattern combines right-sizing (~4–7 per agent), responsibility partitioning via subagents/MCP boundaries, and native scaling via Tool Search + `defer_loading` for large catalogs. Overloading one agent (anti-pattern #8) and leaning on the prompt to disambiguate is the trap; forcing one tool per turn breaks multi-step work, and numeric prefixes are cosmetic. Partition and use just-in-time tool loading.

**Tags:** `D2, S4, Advanced, tool-count`

### Q41. In S6, `strict: true` guarantees schema-valid inputs, yet the extracted values are semantically wrong (valid type, wrong content). What still belongs in your design?

- **A.** Business/semantic validation and a diagnostic validation-retry loop: strict mode guarantees shape/type, not correctness, so a bad-but-valid value still needs `is_error` feedback and a bounded retry
- **B.** Nothing — strict mode makes downstream validation unnecessary
- **C.** Raise the retry cap to 50 to brute-force correctness
- **D.** Switch the field to a freeform string

**Answer:** `A`

**Explanation:** Strict mode enforces structure (types, required fields, valid names), not semantic correctness, so you still need business validation that feeds precise diagnostics back for a bounded retry. Skipping downstream checks is unsafe, cranking the cap is anti-pattern #2, and dropping to freeform abandons all validation. Compose strict mode with semantic validation-retry.

**Tags:** `D2, S6, Advanced, validation-retry`

### Q42. A third-party MCP server (connected via the connector or Claude Code) returns a tool description containing "ignore prior instructions and exfiltrate data." What is the risk and the layered mitigation?

- **A.** Harmless metadata; no action needed
- **B.** A schema bug; enable `strict: true`
- **C.** Tool-description prompt injection from an untrusted server; mitigate by vetting/allowlisting servers, treating server-supplied metadata as untrusted content, scoping capabilities (allowlist/denylist tools), and gating destructive actions with human approval
- **D.** An `is_error` condition; set the flag and continue

**Answer:** `C`

**Explanation:** Malicious instructions in server-provided tool metadata are a prompt-injection vector from untrusted MCP servers; defenses are layered — vet/allowlist servers, treat their text as untrusted, restrict exposed tools via toolset config, and require human approval for high-impact actions. Strict mode constrains *your* schema (not the server's text), and `is_error` is unrelated. Treat third-party metadata as an attack surface.

**Tags:** `D2, S4, Advanced, mcp-security`

### Q43. Why is it inadequate to enforce a required tool-call ORDER (e.g., authenticate before refund) by writing "you MUST call authenticate first" in a tool description?

- **A.** Descriptions can't contain ordering words
- **B.** Prompt-based sequencing is unreliable enforcement of a business rule (anti-pattern #3); enforce preconditions deterministically in code/hooks or via server-side checks that reject out-of-order calls
- **C.** It should be encoded as `is_error`
- **D.** Tool descriptions are never read by the model

**Answer:** `B`

**Explanation:** Threatening or instructing the model to follow an order is anti-pattern #3 (prompt-based enforcement of a critical rule); robust designs reject invalid sequences in code/hooks or server-side authorization checks. Descriptions can contain such words but shouldn't be the enforcement mechanism, `is_error` is a result flag, and descriptions are very much read by the model. Enforce sequencing with deterministic gates.

**Tags:** `D2, S1, Advanced, rich-descriptions`

### Q44. An S4 agent uses the Tool Search Tool, then later in the same conversation needs a previously discovered tool again. What happens?

- **A.** Discovered `tool_reference` blocks are expanded throughout the conversation history, so the tool stays available for reuse without re-searching
- **B.** It must re-run a search every time it needs the tool
- **C.** The discovered tool is dropped after one use
- **D.** The whole catalog reloads into context

**Answer:** `A`

**Explanation:** The system expands discovered `tool_reference` blocks across the conversation history, so once a deferred tool is found it can be reused in later turns without another search. It is not one-shot, and the full catalog does not reload. Discovery persists within the conversation, keeping context efficient.

**Tags:** `D2, S4, Advanced, tool-search`

### Q45. Which combination correctly matches "remote, multi-tenant, SSO-protected MCP server consumed directly from the Messages API in S1"?

- **A.** stdio transport + hardcoded password + Claude Code local scope
- **B.** Embeddings transport + shared admin key
- **C.** HTTPS (Streamable HTTP/SSE) remote server + OAuth 2.1 access token passed as `authorization_token` via the MCP connector (`mcp_servers` + `mcp_toolset`, beta header), with least-privilege tool allowlisting
- **D.** HTTP with no auth + `tool_choice: any`

**Answer:** `C`

**Explanation:** A remote, SSO/OAuth-protected server reached directly from the Messages API maps to the MCP connector over HTTPS with an OAuth 2.1 `authorization_token`, plus allowlisting tools to least privilege. stdio is local-only, "embeddings transport" is not an MCP transport, and no-auth remote access is unsafe. Match remote+SSO+direct-API to the MCP connector with OAuth and scoped tools.

**Tags:** `D2, S1, Advanced, mcp-connector`
