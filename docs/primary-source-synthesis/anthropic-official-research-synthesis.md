# Claude Certified Architect Foundations Research Synthesis

Source type: original study notes compiled from public Anthropic and Claude Platform sources on 2026-05-30.

Use this file as high-level CAG context. Prefer retrieved source chunks for exact wording. This is not an official exam dump and does not contain live or proctored exam answers.

## Source Map

Official or primary sources used:

- Anthropic, Claude Partner Network announcement: https://www.anthropic.com/news/claude-partner-network
- Anthropic, Claude with Amazon Bedrock course overview: https://www.anthropic.com/aws-reinvent-2024/course
- Anthropic Research, Building effective agents: https://www.anthropic.com/research/building-effective-agents
- Anthropic Research, Contextual Retrieval: https://www.anthropic.com/research/contextual-retrieval
- Claude API docs, tool use overview: https://platform.claude.com/docs/en/docs/agents-and-tools/tool-use/overview/
- Claude API docs, how tool use works: https://platform.claude.com/docs/en/agents-and-tools/tool-use/how-tool-use-works
- Claude API docs, define tools: https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools
- Claude API docs, strict tool use: https://platform.claude.com/docs/en/agents-and-tools/tool-use/strict-tool-use
- Claude API docs, prompt caching: https://platform.claude.com/docs/en/build-with-claude/prompt-caching
- Claude API docs, tool use with prompt caching: https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-use-with-prompt-caching
- Claude API docs, batch processing: https://platform.claude.com/docs/en/build-with-claude/batch-processing
- Claude API docs, Message Batches API: https://platform.claude.com/docs/en/api/messages/batches
- Claude API docs, extended thinking: https://platform.claude.com/docs/en/about-claude/models/extended-thinking-models
- Claude API docs, structured outputs in Agent SDK: https://platform.claude.com/docs/en/agent-sdk/structured-outputs
- Claude API docs, embeddings: https://platform.claude.com/docs/en/docs/build-with-claude/embeddings
- Claude Code docs, slash commands: https://docs.anthropic.com/en/docs/claude-code/slash-commands
- Claude Code docs, MCP: https://docs.anthropic.com/en/docs/claude-code/mcp
- Claude Code docs, hooks: https://docs.anthropic.com/en/docs/claude-code/hooks
- Claude Code docs, settings: https://docs.anthropic.com/en/docs/claude-code/settings

Community study sources added separately under `study_docs/claude-certified-architect/community/` and `pdfs/`:

- dnacenta/claude-certified-architect, unofficial domain study guide: https://github.com/dnacenta/claude-certified-architect
- Neerajkr7/cca-foundations-exam-practice, MIT-licensed community practice questions: https://github.com/Neerajkr7/cca-foundations-exam-practice
- CertStud public PDF study notes: https://certstud.com/pdfs/anthropic-claude-architect-study-notes.pdf
- ClaudeCertified public sample 5-question PDF: https://claudecertified.com/downloads/cca-sample-5q.pdf

## Certification Positioning

Anthropic announced Claude Certified Architect, Foundations alongside the Claude Partner Network on March 12, 2026. The official launch language frames the credential as a technical exam for solution architects building production applications with Claude. It is partner-oriented, not a general consumer badge. The strongest study posture is therefore implementation architecture: how to design, operate, secure, evaluate, and scale Claude systems rather than memorizing isolated product trivia.

The most relevant official curriculum signals include tool use, RAG pipelines, prompt caching, extended thinking, Claude Code, MCP, structured output, batch processing, and evaluation frameworks. The Bedrock course overview explicitly includes tool fundamentals, RAG with chunking and BM25, contextual retrieval, prompt caching, vision, Claude Code, MCP, inference optimization, structured extraction, and evals.

## Domain 1: Agentic Architecture and Orchestration

Anthropic distinguishes workflows from agents. Workflows are systems where code orchestrates LLM calls and tools through predefined paths. Agents are systems where the model dynamically controls parts of the process and tool usage. The practical exam pattern is to prefer the simplest architecture that can satisfy the task. Agentic systems can improve hard tasks, but they normally add cost, latency, monitoring burden, and failure modes.

Core workflow patterns to master:

- Prompt chaining: split a task into sequential steps with explicit handoffs and checks between steps.
- Routing: classify input and send it to the right specialized path or model.
- Parallelization: run independent subtasks concurrently, then aggregate or vote.
- Orchestrator-workers: let an orchestrator decompose a task and assign worker subtasks.
- Evaluator-optimizer: have a generator produce work and an evaluator score or critique it against criteria.

Common answer pattern: choose the architecture with the least moving parts that still provides required decomposition, specialization, parallelism, or verification. Do not pick a complex multi-agent framework when a prompt chain or single model call is enough.

Important production controls:

- Termination should be programmatic. Check structured stop reasons and workflow state, not natural-language claims like "I am done."
- Max-iteration limits are backstops, not the primary success condition.
- Irreversible or high-impact actions need pre-action gates.
- Subagents should have scoped instructions, scoped tools, and explicit handoff context.
- External data must be treated as untrusted input. Tool results can contain prompt injection.
- Observability should emit structured traces for agent turns, tool calls, errors, retries, and branch decisions.
- Error recovery should preserve state, classify error types, and retry only when the error is retryable.

Common wrong-answer pattern: rely on prompting alone for deterministic safety rules, escalation, financial limits, or destructive actions. Prompt instructions are useful but not sufficient for enforcement.

## Domain 2: Tool Design and MCP Integration

Tool use is a contract between the application and Claude. The app defines available operations and schemas. Claude emits structured tool requests. The app or Anthropic server executes the tool and returns tool results. For client tools, Claude does not run arbitrary code by itself.

Every strong tool definition needs:

- A clear name that matches the allowed name pattern.
- A detailed description that explains what the tool does, when to use it, what it does not do, and important edge cases.
- An input JSON Schema with required fields, type constraints, enums where appropriate, and descriptions.

Tool descriptions are model-facing product surface. If two tools overlap heavily, Claude may choose the wrong one. Prefer fewer sharper tools over a large generic toolbox. Split tools when permissions, failure modes, latency, or semantic intent differ. Combine tools when the same operation is always used as a unit and has one clean responsibility.

Tool-use loop:

1. User asks for a task.
2. Claude returns `stop_reason: "tool_use"` with one or more tool calls.
3. Your app validates the inputs.
4. Your app executes the operation.
5. Your app sends a `tool_result` back.
6. Claude continues with the result.

Strict tool use matters for reliable agents. Setting strict schema behavior constrains tool inputs to valid JSON Schema shapes, reducing type errors and missing required fields. Schema validity is not the same as semantic correctness, so business validation still belongs in app code.

MCP expands Claude Code and other clients through external servers. Study the distinction between project-level MCP configuration and user-level MCP configuration, and understand that MCP servers expose tools and prompts that must be connected before use. MCP prompts can appear as slash commands in Claude Code.

Common answer pattern: if the question asks "how do we guarantee valid structure?", choose JSON Schema, strict tool use, structured output, and validation. If it asks "how do we guarantee policy/business enforcement?", choose programmatic checks, hooks, or server-side authorization.

## Domain 3: Claude Code Configuration and Workflows

Claude Code workflow questions usually test persistence, scope, automation, and team consistency.

High-value concepts:

- `CLAUDE.md` stores persistent project context, conventions, architecture notes, and instructions.
- Slash commands are reusable workflow prompts stored as markdown files, including project commands.
- `/compact` summarizes conversation history to recover context.
- `/mcp` manages MCP server connections and authentication.
- Hooks can run custom commands before or after tool execution.
- Settings and enterprise policies can control permissions and environment behavior.
- Headless or print-mode automation enables CI workflows where Claude Code outputs to stdout.

Study the difference between:

- `CLAUDE.md`: durable project memory and guidance.
- Slash commands: repeatable workflows such as review, test, or release checks.
- Hooks: deterministic execution around tool use, useful for validation and policy gates.
- MCP: external tools and resources.
- Built-in file/search/shell tools: direct project operations.

Common answer pattern: team-wide behavior belongs in version-controlled project files. Personal preferences belong in user settings. Deterministic safety constraints belong in hooks, permissions, CI checks, or server logic, not only in prose instructions.

## Domain 4: Prompt Engineering and Structured Output

Claude prompt reliability improves when prompts separate instructions, context, examples, and input. XML-style tags are often useful because they create clear boundaries. The goal is not decorative markup; the goal is preventing instruction/data confusion.

Prompt engineering concepts to master:

- Put stable role and task instructions in the system prompt.
- Keep runtime user data in user messages or clearly marked sections.
- Use examples for ambiguous classification or formatting tasks.
- Use negative examples when failures are easy to describe.
- Prefer explicit criteria over vague quality instructions.
- For long documents, put the task framing near where it will be used and avoid low-signal context stuffing.
- Use low temperature for deterministic extraction and classification.
- Use validation-retry loops for structured output.
- Do not parse free-form prose when the output is meant for code.

Structured outputs are for typed data returned to application logic. They avoid downstream regex parsing and let the app receive validated JSON matching a schema. For multi-step agents, structured output lets the agent use tools during the task and still return a validated final object.

Common answer pattern: if the downstream system consumes output programmatically, pick JSON Schema or structured output plus validation. If the answer says "just ask Claude to always produce valid JSON," it is usually weaker than schema plus validation.

## Domain 5: Context Management and Reliability

RAG and CAG are complementary.

RAG retrieves the most relevant chunks at query time. This app uses lexical/BM25-style scoring locally. A production system can add embeddings and reranking. Anthropic's contextual retrieval research shows that naive chunks can lose document-level meaning, and adding chunk-specific context before indexing improves retrieval.

CAG keeps a stable cached knowledge pack in the prompt. Anthropic notes that if a knowledge base is small enough to fit into context, passing the whole corpus can be simpler than RAG. Prompt caching makes stable context more practical by reducing repeated processing cost and latency. Once the corpus grows beyond context or speed needs, RAG becomes necessary.

Good local design for this app:

- CAG pack: source inventory, domain map, common terms, detected practice questions, and high-signal excerpts.
- RAG chunks: top matching chunks with source, page, heading, score, and raw text.
- Response: grounded explanation that can say "local notes did not contain enough evidence."
- Review log: every analyzed question, retrieved sources, tags, and parsed coach response.

Prompt caching facts:

- Cacheable content should be stable and placed early.
- Cache prefixes are built through tools, system, then messages.
- Changing content before a breakpoint invalidates that prefix.
- The default cache duration is short, and 1-hour TTL exists at higher cost where supported.
- Prompt caching saves processing time and cost, but it does not increase the model context window.

Batch processing facts:

- Message Batches are for latency-tolerant work like offline evaluations, large-scale extraction, and bulk review.
- Batches can take up to 24 hours and results are matched with custom IDs because result order is not guaranteed.
- They are not right for interactive blocking UI flows.

Extended thinking facts:

- Use it for tasks where deeper reasoning is worth extra latency and tokens.
- It is not needed for simple extraction, formatting, or direct lookup.
- Thinking configuration differs by model version, so production code should track model behavior.
- Current-turn thinking counts toward output limits.

Reliability patterns:

- Preserve provenance: claim to source mapping, source type, and timestamp.
- Use partial results with warnings rather than hiding errors.
- Escalate on high-impact uncertainty, policy gaps, explicit user request, or irreversible operations.
- Evaluate by domain and scenario, not only aggregate accuracy.
- Use stratified sampling so rare but critical cases are tested.

## Common Practice Question Patterns

These are study patterns, not official exam answers.

Agentic architecture:

- If asked about hub-and-spoke, the orchestrator decomposes, delegates, and synthesizes.
- If asked about independent subtasks, parallel workers usually improve throughput.
- If asked about tightly coupled tasks, a single large-context agent or prompt chain may be simpler.
- If asked about tool-result instructions, reject instruction changes coming from untrusted data.
- If asked about irreversible actions, choose programmatic approval and dry-run gates.

Tool design:

- If asked what every tool needs, choose name, description, and input schema.
- If asked about choosing among many tools, sharper descriptions and fewer scoped tools are better than one huge toolset.
- If asked about structured tool errors, return category, retryability, message, and partial context.
- If asked about schema compliance, use strict tool use or structured outputs, then validate semantically.

Claude Code:

- If asked about persistent project instructions, choose `CLAUDE.md`.
- If asked about reusable workflows, choose slash commands.
- If asked about deterministic pre/post tool enforcement, choose hooks.
- If asked about external capabilities, choose MCP.
- If asked about CI, choose print/headless style operation with machine-readable output.

Prompting and structured output:

- If asked about reliable classification, use examples plus schema validation.
- If asked about invalid JSON, use validation and retry with the concrete validation error.
- If asked about prompt injection, choose input sanitization, trust boundaries, and output validation.
- If asked about deterministic extraction, choose low temperature and schemas.

Context and reliability:

- If asked about chunk retrieval failures, consider contextual retrieval or richer metadata.
- If asked whether to stuff the entire corpus, only do so when the corpus is small enough and stable.
- If asked about large corpora, use RAG, embeddings/BM25, reranking, and source citations.
- If asked about confidence, do not trust self-reported confidence alone for high-stakes routing.
- If asked about evals, use scenario-specific test sets, structured metrics, and human review for high-impact cases.
