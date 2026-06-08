# CCA-F Glossary — Key Terms for the Claude Certified Architect (Foundations) Exam

This is an alphabetized reference of every key term a CCA-F candidate must know. Each entry gives a precise definition and, where useful, its exam relevance. It is original study material grounded in publicly documented Anthropic product behavior; it is not an exam dump. Primary sources: platform.claude.com/docs, docs.anthropic.com/en/docs/claude-code, modelcontextprotocol.io, and Anthropic engineering research (anthropic.com/research, anthropic.com/engineering).

## A

### ACI (Agent–Computer Interface)
The design surface between an agent and its tools — tool names, descriptions, input schemas, and result formats. Just as a good HCI (human–computer interface) makes software usable, a good ACI makes tools usable by the model. Exam signal: questions framing "the tool keeps getting misused" usually have a correct answer about improving the ACI (clearer description, better schema, examples) rather than adding prompt instructions.

### Agent SDK (Claude Agent SDK)
Anthropic's framework for building production agents on the same harness that powers Claude Code: it provides the agentic loop, tool execution, context management/compaction, subagents, permissions, and MCP client support. Exam signal: S1 (Customer Support) and S3 (Multi-Agent Research) often hinge on knowing the SDK supplies the loop and orchestration so you do not hand-roll termination or context handling.

### agentic loop
The cycle of: model decides → calls a tool → receives a `tool_result` → reasons again → repeats until it stops. Termination is governed by the API `stop_reason`, not by parsing the model's prose. Exam signal: the single most-tested pattern; the correct loop control checks `stop_reason == "end_turn"` (no more tool calls), with an iteration cap only as a safety backstop.

### augmented LLM
The basic building block of agentic systems: an LLM enhanced with retrieval, tools, and memory. Anthropic's "Building Effective Agents" frames every workflow and agent as composed of augmented LLMs. Exam signal: distinguishes a plain completion from an agent; appears in "which is the simplest sufficient architecture" questions.

## B

### batch API (Message Batches)
An asynchronous endpoint for submitting many Messages requests at once at reduced cost, returning results within a target window rather than synchronously. Best for high-volume, non-latency-sensitive work (offline evals, bulk extraction, CI review fan-out). Exam signal: S5 (CI/CD) and S6 (extraction) — choose batch when throughput/cost matters and you do not need an immediate response.

### breakpoint (cache breakpoint)
A point in the prompt marked with `cache_control` that ends a cacheable prefix. Everything before a breakpoint can be served from cache on later requests if it matches exactly. Exam signal: place breakpoints after large, stable content (system prompt, tool definitions, big documents) and before the volatile tail.

## C

### cache_control / cache breakpoint
The request field (`"cache_control": {"type": "ephemeral"}`) attached to a content block to mark the end of a prefix Claude should cache. Caching is prefix-based: a hit requires the bytes before the breakpoint to be identical. Exam signal: order content stable-to-volatile so the cacheable prefix is maximized; never put changing data ahead of stable data.

### CALM (Context-Aware Loop Management) / context-aware loop control
Catch-all for managing the agentic loop using real signals (stop_reason, tool outcomes, structured state) instead of brittle proxies. Exam signal: the "right pattern" framing behind anti-patterns 1 and 2 — drive the loop with API signals, not natural-language "I'm done" or a hard iteration cap.

### chain-of-thought (CoT)
Prompting the model to reason step by step before answering, improving accuracy on complex tasks. With Claude this is often elicited explicitly or via extended thinking. Exam signal: distinguish ordinary CoT (in the visible output) from extended thinking (a separate, budgeted reasoning channel).

### CLAUDE.md
A special Markdown file Claude Code automatically loads into context as persistent project memory — conventions, commands, architecture notes, and rules. Lives at project root (and can be nested per-directory or in `~/.claude/CLAUDE.md` for global scope). Exam signal: S2 — put durable project rules here; it is the right place for repo conventions, not ad-hoc reminders in each prompt.

### compaction
Automatic summarization of older conversation history when context approaches the window limit, preserving key facts while freeing tokens so a long-running agent can continue. Exam signal: D5 reliability — compaction is the built-in defense against context exhaustion; contrast with naive truncation that drops important state.

### contextual retrieval
An Anthropic technique that prepends a short, chunk-specific context blurb to each chunk before embedding/indexing, drastically reducing failed retrievals versus naive chunking. Pairs well with reranking. Exam signal: the correct fix for "RAG retrieves the wrong chunk" in knowledge-base scenarios.

### context rot
Degradation of model performance as irrelevant or stale content accumulates in the context window — the model gets distracted or confused by noise. Exam signal: D5 — the rationale for compaction, scoping subagents, and curating context; "more context" is not always better.

## E

### elicitation (MCP)
An MCP capability where a server can, mid-operation, request additional structured input from the user via the client. Lets a tool pause and ask for missing parameters or confirmation instead of failing. Exam signal: D2 — recognize elicitation as the protocol-native way to gather missing input rather than guessing or erroring out.

### evaluator-optimizer
A workflow where one LLM generates a result and a second evaluates it against criteria and returns feedback, looping until the evaluation passes. Effective when you have clear evaluation criteria and iterative refinement helps. Exam signal: D1 — choose this for "draft then critique" tasks; the evaluator should ideally be a fresh context to avoid self-review bias (anti-pattern 9).

### extended thinking
A mode where Claude produces internal reasoning in a separate, token-budgeted thinking block before its final answer, improving performance on hard reasoning, math, and multi-step tasks. The thinking budget is configurable. Exam signal: D4/D5 — enable for genuinely hard reasoning; it is distinct from prompting "think step by step" in the answer and interacts with tool use and caching.

## F

### few-shot / multishot prompting
Including example input/output pairs in the prompt to demonstrate the desired format and behavior. Multishot (several diverse examples) sharply improves consistency and adherence to structure. Exam signal: D4 — the standard fix for inconsistent formatting or edge-case handling; examples should be diverse and cover edge cases.

## H

### headless mode
Running Claude Code non-interactively (e.g., `claude -p "<prompt>"` with `--output-format`), suitable for scripts, hooks, and CI pipelines. Exam signal: S5 — the mechanism for invoking Claude Code in CI/CD; combine with structured output formats for machine-readable results.

### hooks
Deterministic shell commands Claude Code runs at defined lifecycle events (e.g., PreToolUse, PostToolUse, Stop) to enforce rules, run formatters/linters, block actions, or inject context — independent of the model's cooperation. Exam signal: anti-pattern 3 — for critical business rules use a hook (code that always runs), not prompt pleading the model may ignore.

### hub-and-spoke
An orchestration topology where a central coordinator (hub) delegates to and aggregates from peripheral subagents (spokes) that do not talk to each other directly. Synonymous in practice with orchestrator-workers. Exam signal: S3 — the canonical multi-agent research shape; the coordinator owns synthesis and the subagents own scoped subtasks.

## I

### is_error
The boolean field on a `tool_result` content block that marks the tool call as failed, letting the model see and react to the error. Exam signal: anti-patterns 6 and 7 — return real diagnostic detail with `is_error: true`; never swallow errors or return empty results as if successful.

## J

### JSON schema / input_schema
The JSON Schema object declaring a tool's parameters (`input_schema` in a tool definition). It constrains and documents the arguments the model must produce. Exam signal: D2/D6 — a tight, well-described schema is the primary lever for correct tool calls; required fields, enums, and descriptions do real work.

## L

### LLM-as-judge
Using a (often separate) model invocation to score or grade outputs against a rubric, for evals or runtime quality gates. Exam signal: D4/D5 — preferred over self-reported confidence (anti-pattern 4); a judge with explicit criteria and a fresh context is more reliable than the author grading itself.

## M

### MCP (Model Context Protocol)
An open standard (modelcontextprotocol.io) for connecting LLM applications to external tools, data, and prompts via a client–server protocol, so integrations are reusable across hosts. Exam signal: D2 is built on MCP; know it standardizes the integration boundary so you do not bespoke-wire every tool.

### MCP primitives (tools / resources / prompts / roots / sampling)
The protocol's core capabilities. **Tools**: model-invokable actions/functions. **Resources**: readable data/context the server exposes (often app-controlled). **Prompts**: reusable, user-selectable prompt templates. **Roots**: client-declared filesystem/URI boundaries scoping where a server may operate. **Sampling**: a server requesting the client/host to run an LLM completion on its behalf. Exam signal: match the primitive to the need — expose read-only data as a resource, an action as a tool, a templated workflow as a prompt; sampling and roots distinguish strong distractors.

## O

### orchestrator-workers
A workflow where a lead model dynamically breaks a task into subtasks, dispatches them to worker LLMs, and synthesizes the results — used when subtasks cannot be predetermined. Exam signal: D1/S3 — choose over fixed parallelization when the decomposition is dynamic; the orchestrator must handle worker errors explicitly, not silently.

## P

### parallelization
Running multiple LLM calls concurrently, either by **sectioning** (independent subtasks run in parallel) or **voting** (same task run several times and aggregated). Exam signal: D1 — pick parallelization for independent subtasks needing speed or for consensus; contrast with prompt chaining (sequential dependency).

### permission modes
Claude Code's controls over how much the agent can do without asking — e.g., default (prompts for risky actions), auto-accept edits, plan mode (read-only), and bypass/YOLO-style modes that skip confirmations. Exam signal: S2/S4 — match the mode to risk; plan mode for safe exploration, tighter modes in untrusted or production-adjacent contexts.

### plan mode
A Claude Code mode where the model can read and analyze but cannot edit files or run mutating actions until you approve a plan it presents. Exam signal: S2 — the correct first step for large or risky changes; it produces a reviewable plan before any write occurs.

### prefill
Seeding the start of the assistant's response with text you supply, steering format or skipping preamble (e.g., prefilling `{` to force JSON, or a tag to enforce structure). Exam signal: D4 — a lightweight way to enforce output shape; note prefill interacts with tool use and is not allowed with extended thinking.

### prompt caching
Reusing a previously processed prompt prefix to cut latency and cost on repeated calls; matching is exact-prefix and controlled with `cache_control` breakpoints. Exam signal: D5/S1 — structure prompts stable-to-volatile (system + tools + long context first) so the cacheable prefix is large and hits often.

### prompt chaining
Decomposing a task into a fixed sequence of LLM calls where each step's output feeds the next, optionally with programmatic gates between steps. Exam signal: D1 — the right choice when subtasks have a clear, dependent order; trades latency for accuracy and is simpler than a full agent.

## R

### RAG (Retrieval-Augmented Generation)
Fetching relevant external documents at query time and supplying them to the model as context, rather than relying on parametric memory. Exam signal: S4 — combine with contextual retrieval and reranking; the failure mode is retrieving the wrong context, fixed at the retrieval layer, not by scolding the model.

### reranking
A second-stage scoring pass that reorders an initial set of retrieved candidates by relevance (e.g., a cross-encoder/rerank model), so the top-k actually fed to the model is higher quality. Exam signal: pairs with contextual retrieval as the standard recipe to raise retrieval precision.

### routing
A workflow that classifies an input and directs it to a specialized prompt, tool, or model best suited to it (e.g., easy queries to Haiku, hard ones to Opus). Exam signal: D1 — choose routing when inputs fall into distinct categories with different handling; classification should be deterministic where possible.

## S

### slash commands
Reusable, named Claude Code commands (built-in like `/clear`, or custom Markdown files under `.claude/commands/`) that expand into a predefined prompt/workflow, optionally taking arguments. Exam signal: S2 — encapsulate repeatable workflows as project slash commands instead of re-typing long prompts.

### stop_reason
The field on a Messages API response indicating why generation stopped: `end_turn` (model finished), `tool_use` (model wants a tool — continue the loop), `max_tokens`, `stop_sequence`, `pause_turn`, or `refusal`. Exam signal: the authoritative loop-control signal — `tool_use` means run the tool and loop; `end_turn` means stop. Driving the loop off prose instead is anti-pattern 1.

### strict / structured outputs
Mechanisms that constrain Claude to emit output conforming exactly to a declared schema — via `tool_use` with an `input_schema`, structured-output settings, or prefill — yielding reliably parseable JSON. Exam signal: D4/S6 — prefer schema-enforced output over free-text-then-parse; pair with validation-retry for guaranteed-valid results.

### streamable HTTP vs stdio vs SSE (MCP transports)
How an MCP client and server communicate. **stdio**: local subprocess over standard in/out — simplest for local servers. **SSE (Server-Sent Events)**: an older remote transport, now largely superseded. **Streamable HTTP**: the current remote transport supporting bidirectional streaming over HTTP. Exam signal: D2 — pick stdio for local/embedded servers and streamable HTTP for remote/hosted ones; SSE is the legacy option.

### subagent
A scoped, often parallel agent spawned by a coordinator (in Claude Code or the Agent SDK) with its own fresh context window and a narrow toolset/task. Exam signal: S3/S4 — the right tool for isolating context, splitting an overloaded toolset (anti-pattern 8), and getting unbiased review (fresh context counters anti-pattern 9).

### system prompt altitude
Pitching system-prompt instructions at the right level of abstraction — high enough to generalize and not over-constrain, specific enough to be actionable; avoid brittle, hard-coded edge-case rules. Exam signal: D4 — "right altitude" answers favor durable guidance and heuristics over exhaustive if/else lists baked into the prompt.

## T

### tool_choice
The request control over whether/which tool the model uses: `auto` (model decides), `any` (must use some tool), `tool` (must use a named tool), or `none`. Exam signal: D2/S6 — force a specific tool with `tool` to guarantee structured extraction; use `auto` for open-ended agentic work.

### tool_use / tool_result
The paired content blocks of function calling: the model emits a `tool_use` block (name + input) and you return a `tool_result` block (same `tool_use_id`, the output, and `is_error` if it failed). Exam signal: the mechanical core of the agentic loop and structured extraction; correct id pairing and honest error signaling are tested.

### validation-retry
The pattern of validating tool/model output against a schema or business rules and, on failure, feeding the specific validation error back so the model corrects itself — looping until valid or a backstop is hit. Exam signal: S6 — the right way to guarantee valid structured data; the retry prompt must include the concrete error, not a generic "try again."
