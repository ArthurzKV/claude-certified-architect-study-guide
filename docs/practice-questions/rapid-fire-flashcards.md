# CCA-F Rapid-Fire Recall Flashcards

Dense, high-yield recall cards for the **Claude Certified Architect – Foundations (CCA-F)** exam. Each card is a single question (ending in "?") followed by a concise `Answer:` and a one-line `Why:`. Use these for spaced repetition across all five domains. These are original study notes grounded in publicly documented Anthropic behavior — not live exam content.

Sources referenced throughout: platform.claude.com/docs, docs.anthropic.com/en/docs/claude-code, modelcontextprotocol.io, and Anthropic engineering posts (e.g., "Building effective agents", the multi-agent research system writeup).

---

## Exam Format & Logistics Flashcards

### F1. How many questions are on the CCA-F exam and how long is the sitting?

**Answer:** 60 multiple-choice, scenario-based questions in 120 minutes.

**Why:** Roughly two minutes per question; pacing matters because every question requires reading a scenario.

### F2. What is the passing score and the score scale for CCA-F?

**Answer:** 720 out of 1000, on a scaled range of 100–1000.

**Why:** Scaled scoring means raw-to-scaled conversion varies by form; aim well above 720 to be safe.

### F3. Is there a penalty for guessing on CCA-F?

**Answer:** No penalty for guessing — always answer every question.

**Why:** Leaving blanks can only lose points; an educated guess has positive expected value.

### F4. How many production scenarios exist and how many appear in a single sitting?

**Answer:** 6 scenarios exist; 4 are randomly selected per sitting.

**Why:** Every question is anchored to one of the 4 selected scenarios, so you must know all 6.

### F5. Who is the target audience for CCA-F?

**Answer:** Solution architects with 6+ months of hands-on Claude experience.

**Why:** Questions assume practical familiarity, not just doc recall.

### F6. What are the five CCA-F domains and their weights?

**Answer:** D1 Agentic Architecture & Orchestration 27%, D2 Tool Design & MCP 18%, D3 Claude Code Config & Workflows 20%, D4 Prompt Engineering & Structured Output 20%, D5 Context Management & Reliability 15%.

**Why:** D1 is the single largest slice — orchestration and stopping logic dominate.

### F7. What technologies are under test on CCA-F?

**Answer:** Claude Code, Claude Agent SDK, the Claude API (Messages, tool use, prompt caching, batch, extended thinking, structured outputs, embeddings), and MCP.

**Why:** The exam spans the full developer surface, not just one product.

### F8. What are the six named CCA-F production scenarios?

**Answer:** S1 Customer Support Resolution Agent, S2 Code Generation with Claude Code, S3 Multi-Agent Research System, S4 Developer Productivity with Claude, S5 Claude Code for CI/CD, S6 Structured Data Extraction.

**Why:** Knowing each scenario's tech focus tells you which patterns a question expects.

---

## Domain 1 — Agentic Architecture & Orchestration Flashcards

### F9. What is the single correct signal for an agent loop to know a turn is complete?

**Answer:** The API `stop_reason` field on the response.

**Why:** It is the structured, deterministic source of truth — never parse prose like "I'm done."

### F10. What `stop_reason` value tells your loop the model wants to call a tool?

**Answer:** `tool_use`.

**Why:** Your harness must run the tool, append a `tool_result`, and call the API again to continue the loop.

### F11. What `stop_reason` indicates the model finished its turn normally?

**Answer:** `end_turn`.

**Why:** That is the natural completion signal; combined with no pending tool calls it means the task turn is done.

### F12. What `stop_reason` means the response hit the token limit?

**Answer:** `max_tokens`.

**Why:** The output was truncated — you must continue or raise the limit, not treat it as a clean finish.

### F13. What `stop_reason` appears when a stop sequence is hit?

**Answer:** `stop_sequence`.

**Why:** The model emitted a configured stop string; useful for bounded structured output.

### F14. What `stop_reason` is returned when the model declines for safety/policy reasons?

**Answer:** `refusal`.

**Why:** Treat it distinctly from `end_turn` — the content was not a normal completion.

### F15. Is an iteration cap a valid PRIMARY loop-termination mechanism?

**Answer:** No — it is a safety backstop only.

**Why:** The primary control is checking `stop_reason`/no-more-tool-calls; caps just prevent runaway loops (anti-pattern #2).

### F16. When should you enforce a critical business rule — in the prompt or in code?

**Answer:** In deterministic code/hooks, not by pleading in the prompt.

**Why:** Prompt-based enforcement is probabilistic; critical rules need guaranteed execution (anti-pattern #3).

### F17. Should self-reported confidence scores drive escalation decisions?

**Answer:** No — confidence scores are unreliable for control flow.

**Why:** Models are poorly calibrated on self-confidence; use deterministic signals like tool failures or explicit policy (anti-pattern #4).

### F18. Should customer sentiment trigger escalation to a human?

**Answer:** No — sentiment ≠ task complexity or resolvability.

**Why:** Escalate on capability/authority boundaries (refunds over a threshold, missing tools), not on tone (anti-pattern #5).

### F19. When is a single-agent loop preferable to a multi-agent system?

**Answer:** When the task is well-scoped, mostly sequential, and fits one curated tool set in one context window.

**Why:** Multi-agent adds token cost and coordination overhead; only use it when parallel exploration or context isolation pays off.

### F20. What is the coordinator/orchestrator-worker pattern in a multi-agent system?

**Answer:** A lead agent decomposes the task and spawns subagents that work in parallel, then synthesizes their results.

**Why:** It is the canonical pattern for S3 research systems — fan-out then aggregate.

### F21. Why give each subagent its own context window?

**Answer:** To isolate context, run in parallel, and avoid polluting the coordinator's context with raw intermediate detail.

**Why:** Subagents return distilled findings; this is how multi-agent systems scale beyond one window (S3).

### F22. Roughly how much more token usage do multi-agent systems consume vs. single-agent chat?

**Answer:** Substantially more — Anthropic reported multi-agent research used about 15x the tokens of plain chat.

**Why:** You trade tokens for breadth/parallelism, so reserve multi-agent for high-value tasks (S3).

### F23. What is the prompt-chaining workflow pattern?

**Answer:** Decompose a task into a fixed sequence of LLM calls where each step's output feeds the next, optionally with a gate between steps.

**Why:** Best when the task has clean sequential subtasks and you want to trade latency for reliability.

### F24. What is the routing workflow pattern?

**Answer:** Classify an input, then direct it to a specialized prompt/tool/model path.

**Why:** Lets you optimize each category separately (e.g., route easy tickets to Haiku, hard ones to a richer pipeline) (S1).

### F25. What is the parallelization (sectioning/voting) pattern?

**Answer:** Run independent subtasks concurrently (sectioning) or run the same task multiple times and aggregate (voting).

**Why:** Sectioning splits work; voting boosts reliability on a single hard judgment.

### F26. What is the evaluator-optimizer loop?

**Answer:** One call generates, a separate call evaluates against criteria, and feedback loops until the criteria are met.

**Why:** Effective when you have clear evaluation signals and iterative refinement helps quality.

### F27. What is the difference between a "workflow" and an "agent" in Anthropic's framing?

**Answer:** Workflows have predefined code paths orchestrating LLM calls; agents dynamically direct their own process and tool use.

**Why:** Anthropic's guidance: start with the simplest thing (often a workflow) and only add agentic autonomy when needed.

### F28. What is the first rule of building effective agents per Anthropic?

**Answer:** Find the simplest solution possible and only increase complexity when it demonstrably improves outcomes.

**Why:** Agents add latency, cost, and failure modes — don't reach for them by default.

### F29. In an agent loop, after a `tool_use` block, what must the next user-turn message contain?

**Answer:** A `tool_result` content block referencing the matching `tool_use_id`.

**Why:** The model needs the tool's output to continue; mismatched/missing IDs break the loop.

### F30. How should you handle a tool error so the agent can recover?

**Answer:** Return a `tool_result` with `is_error: true` and a descriptive message the model can act on.

**Why:** Hiding errors or returning fake empty success blinds the model (anti-patterns #6 and #7).

### F31. What is "human-in-the-loop" used for in agent design?

**Answer:** Gating high-risk or irreversible actions behind explicit human approval.

**Why:** It is a deterministic safety control for actions like refunds, deletions, or production deploys (S1, S5).

### F32. Why are checkpoints/state persistence important in long agent runs?

**Answer:** They let you resume after failure and inspect progress without rerunning everything.

**Why:** Long multi-step agents are fragile; durable state turns crashes into resumable steps (S3).

---

## Domain 2 — Tool Design & MCP Integration Flashcards

### F33. What is the recommended number of tools to expose to a single agent?

**Answer:** A tight, curated set of roughly 4–7 tools.

**Why:** Too many tools (e.g., 18) degrade selection accuracy; split via subagents or MCP boundaries (anti-pattern #8).

### F34. How do you scale beyond the ideal tool count without overloading one agent?

**Answer:** Partition tools across subagents or behind separate MCP servers, each agent seeing only its relevant subset.

**Why:** Each agent keeps a focused tool surface, improving selection accuracy.

### F35. What three fields define a tool in the Claude API tool schema?

**Answer:** `name`, `description`, and `input_schema` (a JSON Schema for the inputs).

**Why:** The description and schema are the model's only guide to when/how to call the tool — invest in them.

### F36. What makes a good tool description for Claude?

**Answer:** Clear purpose, when to use vs. not use, parameter meanings, and expected return — written for the model as the audience.

**Why:** Tool-selection errors usually trace back to vague descriptions, not the model.

### F37. What does MCP stand for and what problem does it solve?

**Answer:** Model Context Protocol — an open standard for connecting LLM apps to external tools, data, and prompts via a uniform interface.

**Why:** It standardizes integrations so any MCP client can use any MCP server ("USB-C for AI").

### F38. What are the three core MCP server primitives?

**Answer:** Tools, Resources, and Prompts.

**Why:** Tools are model-invoked actions, Resources are app/data context, Prompts are user-invoked templated workflows.

### F39. What is the key distinction between an MCP Tool and an MCP Resource?

**Answer:** Tools are model-controlled (the model decides to invoke them); Resources are application/data-controlled context exposed to the client.

**Why:** Resources provide read context; Tools perform actions — exam loves this control distinction.

### F40. What is an MCP Prompt primitive?

**Answer:** A reusable, often user-invoked templated message/workflow the server exposes (e.g., a slash-command-style prompt).

**Why:** Prompts are typically user-controlled entry points, distinct from model-controlled Tools.

### F41. What are the standard MCP transports?

**Answer:** stdio (local subprocess) and Streamable HTTP (remote); Streamable HTTP superseded the older HTTP+SSE transport.

**Why:** Pick stdio for local servers, Streamable HTTP for networked/remote servers.

### F42. Which MCP transport is preferred for a local server running on the same machine?

**Answer:** stdio.

**Why:** It launches the server as a subprocess over standard in/out — simple and low-latency for local tools (S4).

### F43. Which MCP transport is used for remote/hosted servers today?

**Answer:** Streamable HTTP.

**Why:** It replaced the legacy HTTP+SSE transport for remote connectivity in the current spec.

### F44. What underlying message format does MCP use?

**Answer:** JSON-RPC 2.0.

**Why:** Requests, responses, and notifications all follow JSON-RPC over the chosen transport.

### F45. In MCP terms, what are the "host," "client," and "server"?

**Answer:** Host is the LLM app (e.g., Claude Desktop/Code), the client lives in the host and maintains one connection per server, and the server exposes tools/resources/prompts.

**Why:** One client connects to one server; a host can run many clients.

### F46. What is "tool poisoning" or a malicious MCP server risk?

**Answer:** A server can embed harmful instructions in tool descriptions/results to manipulate the model (prompt injection via tools).

**Why:** Only connect trusted MCP servers and review tool descriptions — security is an architect concern.

### F47. When designing a tool that can fail, what should the error result include?

**Answer:** Actionable diagnostic detail (what failed, why, how to fix) the model can use to retry or adjust.

**Why:** Generic "an error occurred" messages hide context the model needs (anti-pattern #6).

### F48. Should a tool ever return empty results to mask an internal failure?

**Answer:** No — that silently swallows errors and looks like a successful empty answer.

**Why:** The model then "succeeds" on wrong data; surface the error instead (anti-pattern #7).

### F49. How do you add a remote MCP server to Claude Code from the CLI?

**Answer:** `claude mcp add` (with transport and URL/command), then it appears in the session's tool list.

**Why:** Claude Code reads MCP config and exposes the server's tools to the agent (S4).

### F50. What is "namespacing" of MCP tools and why does it matter?

**Answer:** Tools are surfaced with their server prefix so names don't collide across servers.

**Why:** Prevents ambiguity when multiple MCP servers expose similarly named tools.

---

## Domain 3 — Claude Code Configuration & Workflows Flashcards

### F51. What is CLAUDE.md and what does Claude Code do with it?

**Answer:** A project memory file Claude Code automatically loads into context at session start.

**Why:** It is the canonical place for build commands, conventions, and constraints — persistent project instructions (S2).

### F52. What belongs in CLAUDE.md vs. a one-off prompt?

**Answer:** Durable, repo-wide facts and conventions (commands, style, do/don't) belong in CLAUDE.md; task-specific asks go in the prompt.

**Why:** CLAUDE.md is reloaded every session, so it should hold stable guidance, not throwaway requests.

### F53. Where can CLAUDE.md files live and how do they combine?

**Answer:** Project root, subdirectories, and a user-level `~/.claude/CLAUDE.md`; they layer together, with more specific/closer files adding to broader ones.

**Why:** Lets you set global preferences plus per-repo and per-subtree rules.

### F54. What is plan mode in Claude Code and when do you use it?

**Answer:** A read-only mode where Claude investigates and proposes a plan before making any edits.

**Why:** Use it for complex or risky changes so you approve the approach before code is written (S2).

### F55. How do you keep Claude Code from editing files while it researches?

**Answer:** Use plan mode (read-only) until you approve the plan.

**Why:** It separates investigation from mutation, reducing premature or wrong edits (S2).

### F56. What is a custom slash command in Claude Code and where is it defined?

**Answer:** A reusable prompt stored as a Markdown file under `.claude/commands/` (project) or `~/.claude/commands/` (user), invoked as `/name`.

**Why:** Encapsulates repeatable workflows (e.g., `/review`, `/fix-ci`) for the team (S2, S5).

### F57. How do slash commands accept dynamic input?

**Answer:** Via argument placeholders (e.g., `$ARGUMENTS`) substituted into the command's prompt at invocation.

**Why:** Lets one command template handle varied inputs like a ticket ID or file path.

### F58. What are Claude Code hooks?

**Answer:** User-defined shell commands that run deterministically at lifecycle events (e.g., PreToolUse, PostToolUse, Stop).

**Why:** They enforce rules (formatting, blocking dangerous commands) in code rather than relying on the prompt (anti-pattern #3).

### F59. Which hook would you use to block a disallowed tool action before it runs?

**Answer:** A PreToolUse hook that inspects the tool call and can deny it.

**Why:** It runs before execution and can veto the action deterministically (S5).

### F60. What is the difference between Claude Code "subagents" and the orchestrator pattern?

**Answer:** Claude Code subagents are configured assistants (own prompt, tools, separate context) the main agent can delegate to.

**Why:** They let you isolate context and curate tools per task within Claude Code (S4).

### F61. What are Claude Code's built-in tools, broadly?

**Answer:** File read/edit/write, shell/Bash execution, search/grep/glob, and web fetch — the core coding toolset.

**Why:** These cover codebase exploration and modification without extra MCP setup (S4).

### F62. How do you run Claude Code non-interactively for CI/CD?

**Answer:** Headless/print mode (`claude -p "..."`), which runs once and prints output, scriptable in pipelines.

**Why:** It enables automated review/fix jobs in CI (S5).

### F63. How do you get machine-parseable output from Claude Code in a pipeline?

**Answer:** Use the JSON output format flag so the run emits structured JSON instead of prose.

**Why:** CI steps can parse results reliably instead of scraping text (S5).

### F64. What are Claude Code permission modes and why do they matter for CI?

**Answer:** Modes control auto-approval of tool actions (e.g., default prompts, plan mode, or auto-accept edits); CI typically pre-allows a constrained set.

**Why:** Pipelines can't answer interactive prompts, so permissions must be scoped up front and safely (S5).

### F65. What is the recommended way to give Claude Code project-specific tool permissions?

**Answer:** Configure allow/deny rules in settings (e.g., `.claude/settings.json`) committed with the repo.

**Why:** Makes permissions reviewable and consistent for the team rather than ad hoc (S5).

### F66. How should you structure a multi-pass code review in CI with Claude Code?

**Answer:** Run separate passes (e.g., correctness, security, style) ideally in fresh contexts, aggregating findings.

**Why:** Separating concerns and contexts avoids one pass's bias contaminating another (anti-pattern #9, S5).

### F67. Why use a fresh context for a review pass instead of reusing the author session?

**Answer:** A same-session reviewer inherits the author's reasoning and rationalizations, missing its own bugs.

**Why:** Independent context yields genuine critique (anti-pattern #9).

---

## Domain 4 — Prompt Engineering & Structured Output Flashcards

### F68. What is the most reliable way to force a specific JSON structure from Claude?

**Answer:** Use tool_use with a tool whose `input_schema` is the desired JSON Schema (and optionally force that tool).

**Why:** The model fills a validated schema rather than free-form JSON in prose, drastically cutting parse errors (S6).

### F69. How do you force Claude to call a particular tool every time?

**Answer:** Set `tool_choice` to `{ "type": "tool", "name": "..." }` (or `any` to force some tool).

**Why:** Guarantees structured extraction output instead of a chatty reply (S6).

### F70. What `tool_choice` options exist and what do they mean?

**Answer:** `auto` (model decides), `any` (must call some tool), `tool` (must call the named tool), and `none` (no tools).

**Why:** Pick `tool`/`any` for deterministic structured extraction; `auto` for genuine agentic decisions.

### F71. What is a robust loop for structured extraction with validation?

**Answer:** Get tool_use output, validate against the schema, and on failure feed the validation error back as a tool_result for the model to correct.

**Why:** A validation-retry loop fixes malformed output deterministically (S6).

### F72. Why prefill or constrain output rather than ask "respond only in JSON" in prose?

**Answer:** Prose instructions are probabilistic; schema-constrained tool_use is structurally enforced.

**Why:** Critical formatting should be enforced by mechanism, not request (S6).

### F73. What is extended thinking and when do you enable it?

**Answer:** A mode where Claude produces internal reasoning (thinking blocks) before answering, enabled via a thinking budget; use it for hard multi-step reasoning.

**Why:** It improves complex analysis but costs tokens/latency — reserve it for genuinely hard tasks.

### F74. Can you combine extended thinking with tool use?

**Answer:** Yes — interleaved thinking lets the model reason between tool calls.

**Why:** Useful for agents that must plan, act, observe, and re-plan (S3).

### F75. When using extended thinking, what must you do with prior thinking blocks in multi-turn tool loops?

**Answer:** Preserve/pass back the thinking blocks (including signatures) as the API requires, rather than stripping them.

**Why:** Dropping required thinking context can break continuity; follow the documented handling.

### F76. What are the canonical Claude prompt-engineering levers, in rough priority order?

**Answer:** Be clear and direct, give examples (multishot), let it think (CoT), use XML tags, give it a role (system prompt), prefill, and chain prompts.

**Why:** This is Anthropic's documented ladder — clarity and examples first, exotic tricks later.

### F77. Why use XML tags in prompts to Claude?

**Answer:** They delimit sections (instructions, context, examples, data) so the model parses structure unambiguously.

**Why:** Claude is specifically tuned to respect XML-style delimiters, reducing instruction bleed.

### F78. What is the role/system prompt used for?

**Answer:** To set Claude's persona, domain, and standing constraints separate from the turn's request.

**Why:** A focused role improves consistency and adherence across a conversation (S1).

### F79. What is multishot (few-shot) prompting and when does it help most?

**Answer:** Including 2+ examples of desired input→output; it most helps formatting consistency and edge-case handling.

**Why:** Examples are often more effective than lengthy instructions for shape and tone.

### F80. What is prefilling the assistant response and what's a caution?

**Answer:** Starting the assistant turn with text to steer format/skip preamble; caution: prefill is not allowed/behaves differently with extended thinking.

**Why:** Powerful for forcing JSON or a leading token, but check mode compatibility.

### F81. How do you reduce hallucination in factual extraction prompts?

**Answer:** Allow "I don't know"/null, ground in provided source text, and require quotes/citations from the source.

**Why:** Giving an explicit out and grounding reduces fabricated fields (S6).

### F82. What is a good way to evaluate prompt quality at scale?

**Answer:** Build an eval set with clear graders and measure per-slice, not just an aggregate average.

**Why:** Aggregate accuracy hides per-category failures (anti-pattern #10).

---

## Domain 5 — Context Management & Reliability Flashcards

### F83. What is prompt caching and what does it optimize?

**Answer:** Caching large, stable prefixes (system prompt, tools, docs) so repeated requests reuse them at reduced cost and latency.

**Why:** It cuts cost dramatically for reused context like big system prompts or tool definitions.

### F84. How do you mark content for prompt caching in the Messages API?

**Answer:** Add a `cache_control` breakpoint (e.g., `{"type": "ephemeral"}`) on the content block up to which you want cached.

**Why:** The breakpoint defines the cached prefix; everything before it is eligible for reuse.

### F85. What ordering rule must you follow to get cache hits?

**Answer:** Put stable content first (tools, system, long docs) and variable content last, since caching matches an exact prefix.

**Why:** Any change before the breakpoint invalidates the cache — stable-to-volatile ordering preserves hits.

### F86. What is the default lifetime of a prompt cache entry, and what extends it?

**Answer:** The default ephemeral TTL is short (about 5 minutes), refreshed on use; a longer TTL option exists for less frequent reuse.

**Why:** Frequent reuse keeps the cache warm; describe behavior, and check current docs for exact pricing/TTL.

### F87. How is a cache write priced relative to a normal input token?

**Answer:** A cache write costs more than a base input token, but cache reads cost a fraction of it.

**Why:** You pay a premium once to write, then save heavily on every hit — worth it for high reuse.

### F88. What is the Batch (Message Batches) API and its main trade-off?

**Answer:** Asynchronous bulk processing of many requests at a large cost discount with higher latency (results within up to 24 hours).

**Why:** Use it for non-urgent, high-volume jobs like nightly CI review or bulk extraction (S5, S6).

### F89. When should you choose Batch over real-time requests?

**Answer:** When throughput and cost matter more than latency and no user is waiting synchronously.

**Why:** Batch trades immediacy for roughly half the cost on large jobs (S5).

### F90. What is the difference between the context window and the output token limit?

**Answer:** The context window is total input+output the model can hold; the output limit caps a single response's generated tokens.

**Why:** You can hit `max_tokens` (output) long before filling the whole window.

### F91. What does context compaction/summarization solve in long agent runs?

**Answer:** It condenses earlier turns into a summary so the agent stays within the window while retaining key state.

**Why:** Prevents context overflow and keeps relevant facts available across many steps (S3).

### F92. Why can stuffing too much into the context window hurt accuracy?

**Answer:** Irrelevant or excessive context dilutes attention and can degrade retrieval of the important parts ("context rot").

**Why:** Curate context — more tokens is not always better.

### F93. What is a good retry strategy for transient API errors (429/5xx)?

**Answer:** Exponential backoff with jitter, respecting rate-limit headers, up to a bounded number of attempts.

**Why:** It recovers from transient failures without hammering the API or hanging forever.

### F94. How should an agent handle a tool that times out or returns an error mid-loop?

**Answer:** Return the error as a `tool_result` with `is_error: true` so the model can retry, choose another tool, or escalate.

**Why:** Surfacing the failure keeps the loop honest instead of silently proceeding (anti-patterns #6, #7).

### F95. What is the right metric granularity when evaluating an extraction pipeline?

**Answer:** Per-document-type / per-field / per-category accuracy, not a single aggregate number.

**Why:** Aggregates mask a class that's failing badly (anti-pattern #10, S6).

### F96. What is idempotency and why does it matter for agent tool calls?

**Answer:** An operation produces the same result if repeated; it makes retries safe (no double charges/duplicate writes).

**Why:** Agents retry, so side-effecting tools should be idempotent or guarded with keys (S1).

### F97. What are embeddings used for in a Claude-centric architecture?

**Answer:** Turning text into vectors for semantic search/retrieval (RAG), clustering, and dedup.

**Why:** They power retrieval that feeds relevant context into the model; note Anthropic recommends third-party embedding providers (e.g., Voyage AI).

### F98. Does Anthropic provide its own first-party embeddings endpoint?

**Answer:** No first-party embeddings model — Anthropic recommends third-party providers such as Voyage AI.

**Why:** Embeddings are sourced externally even in an otherwise Claude-native stack.

---

## Anti-Pattern Recall Flashcards (Classic Wrong Answers)

### F99. What's wrong with stopping an agent loop when the model says "I'm finished"?

**Answer:** It parses natural language instead of the structured `stop_reason` — unreliable.

**Why:** Anti-pattern #1; the model may say it's done while still needing tools, or vice versa.

### F100. Why is "cap the loop at N iterations" a wrong PRIMARY stopping answer?

**Answer:** Caps are a safety backstop, not the control mechanism; the real control is `stop_reason`/no pending tool calls.

**Why:** Anti-pattern #2; relying on a cap means you stop on the wrong condition.

### F101. Why is "add a strong instruction in the prompt to never issue refunds over $X" insufficient?

**Answer:** Prompt instructions are probabilistic; a critical financial rule needs a deterministic hook/code check.

**Why:** Anti-pattern #3; enforce business rules in code, not prose (S1).

### F102. Why not escalate to a human when the model's confidence drops below a threshold?

**Answer:** Self-reported confidence is poorly calibrated and gameable.

**Why:** Anti-pattern #4; escalate on concrete signals (tool failure, policy boundary), not vibes.

### F103. Why is escalating angry customers automatically a poor design?

**Answer:** Sentiment doesn't measure whether the agent can actually resolve the issue.

**Why:** Anti-pattern #5; a calm-but-impossible request still needs escalation, an angry-but-simple one doesn't (S1).

### F104. Why are generic error messages from tools harmful to an agent?

**Answer:** They strip diagnostic context the model needs to recover or retry intelligently.

**Why:** Anti-pattern #6; return what failed and why.

### F105. Why is returning empty results on failure dangerous?

**Answer:** It looks like a successful empty answer, so the model proceeds on false data.

**Why:** Anti-pattern #7; never silently suppress errors.

### F106. Why is giving one agent 18 tools a bad design?

**Answer:** Too many tools degrade selection accuracy; curate ~4–7 and split the rest across subagents/MCP servers.

**Why:** Anti-pattern #8 (S4).

### F107. Why is having the authoring agent review its own output in the same session weak?

**Answer:** It inherits its own reasoning bias and rationalizes mistakes.

**Why:** Anti-pattern #9; use a fresh context/separate reviewer (S5).

### F108. Why is a single aggregate accuracy number a misleading eval?

**Answer:** It can hide a category (e.g., one document type) failing badly.

**Why:** Anti-pattern #10; always evaluate per slice (S6).

---

## Scenario-Anchored Mixed Flashcards

### F109. In the Customer Support agent (S1), what correctly triggers escalation to a human?

**Answer:** Hitting a capability/authority boundary — e.g., an action above a policy threshold or a tool/permission the agent lacks.

**Why:** Deterministic policy boundaries, not sentiment or self-confidence (anti-patterns #4, #5).

### F110. In the Multi-Agent Research System (S3), how do subagents report back to the coordinator?

**Answer:** By returning distilled findings/summaries, not raw dumps, so the coordinator's context stays clean.

**Why:** Context isolation is the point of the orchestrator-worker pattern.

### F111. In Claude Code CI/CD (S5), how do you get reliable pass/fail from a review run?

**Answer:** Run headless with JSON output and gate the pipeline on parsed structured results, using fresh-context passes.

**Why:** Avoids prose scraping and same-session review bias (anti-pattern #9).

### F112. In Structured Data Extraction (S6), what's the correct response to a schema-invalid extraction?

**Answer:** Feed the validation error back to the model via tool_result and retry until valid.

**Why:** A validation-retry loop fixes malformed output deterministically.

### F113. In Code Generation with Claude Code (S2), where do you put the repo's build/test commands?

**Answer:** In CLAUDE.md so every session loads them automatically.

**Why:** It's the persistent project-memory mechanism.

### F114. In Developer Productivity (S4), how do you keep tool selection sharp when adding many external integrations?

**Answer:** Split integrations across separate MCP servers/subagents so each agent sees a focused tool subset.

**Why:** Prevents the 18-tool overload anti-pattern (#8).

### F115. For a high-volume nightly extraction job (S5/S6), which API and caching choices fit?

**Answer:** Batch API for the bulk discount plus prompt caching on the shared system prompt/schema prefix.

**Why:** Cost-optimal for non-urgent, repetitive large workloads.

### F116. Across all scenarios, what is the single most-tested control-flow fact?

**Answer:** Loop termination must key off `stop_reason` (structured signal), with iteration caps only as a backstop.

**Why:** D1 is 27% of the exam and this is its core anti-pattern pair (#1, #2).
