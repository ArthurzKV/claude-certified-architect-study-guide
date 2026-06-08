# Claude API Tool Use Fundamentals (CCA-F D2 / S6 Reference)

Tool use (function calling) is how Claude reaches outside the model to fetch data, run code, or take actions. On the CCA-F exam this is the core of **D2 Tool Design & MCP Integration (18%)** and the backbone of **S6 Structured Data Extraction** and **S1 Customer Support**. Source: platform.claude.com/docs — Agents and tools / Tool use.

## How the Tool-Use Loop Works (stop_reason="tool_use")

You send a Messages API request that includes a `tools` array. When Claude decides to call a tool, the response comes back with `stop_reason: "tool_use"` and one or more `tool_use` content blocks in the assistant message. Each `tool_use` block has an `id`, a `name`, and an `input` object that conforms to the tool's schema.

**Your application — not the API — executes the tool.** The Claude API never runs your code or hits your database; it only emits the request. You run the function, then continue the conversation by appending a new `user` message containing a `tool_result` content block. That block references the original call via `tool_use_id` and carries the `content` (the result string or structured data). You then call the API again with the full message history. Claude reads the result and either calls another tool or returns a final text answer with `stop_reason: "end_turn"`.

The canonical loop is: send request → check `stop_reason` → if `tool_use`, execute and return `tool_result` → repeat → exit when `stop_reason` is `end_turn` (or `max_tokens`/`stop_sequence`).

**Exam signal (D1/D5):** Loop termination is decided by the **API `stop_reason`**, never by parsing natural-language text like "I'm done" or "task complete." Any answer that keys termination off model prose is the classic Anti-Pattern #1 distractor. An iteration cap is a **safety backstop only** (Anti-Pattern #2), not the primary stopping mechanism — the primary signal is `stop_reason != "tool_use"`.

## Defining Tools: name, description, input_schema

Each tool in the `tools` array has three required fields:

- **`name`** — a short identifier (snake_case or camelCase), e.g. `get_weather`. Must be unique within the request and stable across turns.
- **`description`** — a natural-language explanation of what the tool does, when to use it, what it returns, and any constraints or side effects. This is the single most important field for tool-use accuracy.
- **`input_schema`** — a JSON Schema object describing the parameters: types, required fields, enums, and per-property descriptions.

**The description carries the model's behavior.** Empirically, a rich, detailed description (3–4+ sentences covering purpose, parameters, return shape, and edge cases) drives correct tool selection and argument filling far more than schema strictness alone. Treat the description as a mini system prompt for that capability. Per-property `description` strings inside `input_schema` matter too — they tell Claude exactly what each argument means and how to format it.

**Exam signal (D2):** When a question asks "Claude keeps calling the wrong tool / passing bad arguments," the correct fix is almost always **improve the tool descriptions and parameter descriptions**, not crank temperature, not add prompt pleading, not bolt on more tools.

## tool_choice: auto / any / tool / none

The `tool_choice` parameter controls whether and how Claude uses tools:

- **`auto`** (default when tools are present) — Claude decides on its own whether to call a tool or answer directly. Best for conversational agents and support flows where some turns need tools and some don't.
- **`any`** — Claude **must** use one of the provided tools (it cannot emit a plain text answer), but may choose which. Use when every turn must result in a structured action.
- **`tool`** — Claude must call **one specific named tool** (`{"type": "tool", "name": "..."}`). This is the standard trick for **forcing structured output**: define one tool whose schema is your target JSON, set `tool_choice` to that tool, and Claude is guaranteed to return data in that shape.
- **`none`** — Claude cannot use any tool this turn; it must respond with text only.

**Note:** forcing a tool with `any` or `tool` disables Claude's ability to emit chain-of-thought text *before* the call in the same turn, since the response must be the tool call. If you want reasoning before the call, use `auto`.

**Exam signal (S6 / D4):** Structured data extraction that "must always return valid JSON" → define a schema-as-tool and set `tool_choice` to that specific tool. This is preferred over begging in the prompt for JSON (Anti-Pattern #3, prompt-based enforcement of a hard requirement).

## Parallel Tool Use and disable_parallel_tool_use

Claude can request **multiple tool calls in a single turn** — the assistant message contains several `tool_use` blocks at once. This is parallel tool use, and it's a major latency win: independent calls (e.g., look up three accounts) happen in one round trip instead of three. You execute all of them and return **all** corresponding `tool_result` blocks in a single follow-up `user` message.

To force strictly sequential, one-at-a-time calls, set `disable_parallel_tool_use: true` inside the `tool_choice` object. Do this only when calls have ordering dependencies or shared-state hazards (e.g., a write that the next read depends on).

**Exam signal (D1/D5):** A scenario where calls are independent and latency matters → **allow parallel tool use** (the default). A scenario with sequential dependencies or transactional writes → disable it. Don't disable parallelism reflexively; it's a deliberate trade-off.

## Strict / Structured Tool Schemas

By default Claude's tool inputs follow your JSON Schema closely but the API does not hard-guarantee exact conformance for every edge case. **Strict tool use** (structured tool schemas) constrains generation so outputs rigorously match the schema — useful when a downstream parser is brittle and any deviation breaks the pipeline. The trade-off is reduced flexibility and some schema-feature restrictions; enable it when validity matters more than expressiveness.

**Exam signal (S6 / D4):** Pair strict schemas with a **validation-and-retry loop** — validate the returned JSON against your schema in code; on failure, send the validation error back as a `tool_result` so Claude can self-correct. The retry feedback must be specific (the exact validation error), not a generic "that was wrong" (Anti-Pattern #6).

## Token-Efficient Tool Use

Token-efficient tool use is an optimization (opt-in via a beta header on supported models) that reduces the token overhead of tool-call request/response formatting, lowering cost and latency for tool-heavy agents. Conceptually: same loop, fewer tokens spent on the tool-calling machinery. Treat exact header strings and model support as version-dependent — describe the *behavior* (cheaper tool round-trips) rather than memorizing a header you're unsure of.

## Client Tools vs Server Tools

- **Client tools** are the ones **you** define and execute in your own environment — your functions, your APIs, your MCP servers. Claude emits the `tool_use`; your code runs it and returns the `tool_result`. This is the general-purpose path.
- **Server tools** are tools Anthropic runs **on the API side** (e.g., web search, code execution). You enable them in the request, but Anthropic executes them and folds the results back in — you don't run the function yourself.

**Exam signal (D2):** Know the boundary. MCP servers and your business APIs are **client tools** (you own execution). A managed capability like Anthropic-hosted web search is a **server tool** (Anthropic owns execution). Mixing these up is a distractor.

## Error Handling: is_error and Surfacing Diagnostics

When a tool fails, return a `tool_result` with **`is_error: true`** and put a **specific, actionable error message** in the content — the HTTP status, the validation failure, the missing field, the timeout reason. Claude reads this and can retry with corrected arguments, fall back to another tool, or escalate.

**Never do two things:** (1) return a generic "An error occurred" that strips out the diagnostic detail the model needs to recover (Anti-Pattern #6), or (2) silently swallow the failure and return an empty/successful-looking result (Anti-Pattern #7) — that hides breakage and produces confidently wrong downstream behavior.

**Exam signal (D5 / S1 / S3):** The correct answer surfaces **rich diagnostic context** to the model via `is_error` results. Hiding errors or returning empty arrays as if successful is always wrong. Good error context is what lets multi-agent and support systems self-heal instead of failing silently.

## Number of Tools, Naming, and Collisions

Keep each agent's tool set **tight — roughly 4–7 well-curated tools.** Overloading one agent with many tools (e.g., 18) degrades selection accuracy and bloats the context. When you genuinely need broad capability, **split via subagents or MCP server boundaries** so each agent sees only its relevant slice (Anti-Pattern #8).

**Tool names must be unique and non-colliding** within a request. When integrating multiple MCP servers, two servers can expose tools with the same name — namespace/prefix them so Claude isn't ambiguously routing between identical names. Clear, distinct names plus rich descriptions prevent mis-selection.

**Exam signal (D2):** "Agent has 18 tools and picks the wrong one" → reduce and split into subagents / separate MCP boundaries; curate to ~4–7. "Two MCP servers expose `search`" → namespace the names. More tools is not better.

## Chain-of-Thought Before Tool Calls

With `tool_choice: auto`, Claude can produce reasoning **text before** the `tool_use` block in the same turn — a brief "let me think about which tool and arguments" step. This improves tool-selection quality and argument accuracy, especially for multi-step or ambiguous tasks. Extended thinking deepens this further. Forcing a tool (`any`/`tool`) suppresses that pre-call reasoning, so for hard selection problems prefer `auto` plus good descriptions over hard-forcing.

**Exam signal (D4):** When accuracy of *which tool / what arguments* is the problem, enabling reasoning before the call (and improving descriptions) beats forcing a tool or adding more tools.

## Quick Reference: Right Pattern vs Trap

- Terminate loop on **`stop_reason`**, not on model text saying "done." Iteration cap = backstop only.
- Force JSON via **schema-as-tool + `tool_choice` tool**, not by pleading for JSON in the prompt.
- Enforce critical rules in **deterministic code/hooks**, not in prompt instructions.
- Surface failures with **`is_error` + specific diagnostics**, never generic messages or silent empties.
- Keep **~4–7 curated tools**; split overloaded agents into subagents/MCP boundaries; namespace colliding names.
- Use **`auto`** + rich descriptions for selection accuracy; reserve `any`/`tool` for must-act / forced-shape turns.
