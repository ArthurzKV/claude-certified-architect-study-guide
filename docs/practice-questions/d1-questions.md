# Domain 1 — Practice Questions

Domain: Agentic Architecture & Orchestration (27% of the CCA-F exam — the largest bank). These scenario-anchored questions cover workflows vs. agents, the five workflow patterns and how to select them, orchestrator-worker and hub-and-spoke responsibilities, the agent loop, `stop_reason`-based termination, fault tolerance, subagent isolation, human-escalation logic, and cost/latency trade-offs. Reference material: Anthropic "Building effective agents" (https://www.anthropic.com/research/building-effective-agents) and the Claude Agent SDK docs (https://docs.anthropic.com/en/docs/agents-and-tools).

Note on explanations: distractors are referenced by their content (not by option letter) so the rationale stays correct regardless of answer position.

## Workflows vs. Agents — Definitions and Selection

### Q1. According to Anthropic's "Building effective agents," what is the defining difference between a workflow and an agent?

- **A.** Workflows orchestrate LLMs and tools through predefined code paths, while agents let the model dynamically direct its own process and tool use
- **B.** Workflows use Claude and agents use a deterministic rules engine without an LLM
- **C.** Workflows always run faster because they cache every response, while agents never cache
- **D.** Workflows can only run locally while agents must run in the cloud

**Answer:** `A`

**Explanation:** Anthropic distinguishes the two by control flow: workflows follow predefined code paths, while agents dynamically direct their own process and tool usage at runtime. The "rules engine without an LLM" option is wrong because workflows still use LLMs — the difference is who decides the next step, not whether a model is present. This is the foundational framing for every D1 question.

**Tags:** `Domain 1` `Foundational` `Workflows vs Agents`

### Q2. A team can solve a task with a fixed three-step pipeline whose steps never change. Per Anthropic's guidance, which approach should they prefer?

- **A.** An autonomous agent, because agents are always more capable
- **B.** A multi-agent hub-and-spoke system for maximum parallelism
- **C.** A predefined workflow, because the task is predictable and a workflow gives more consistency and lower cost than open-ended agency
- **D.** A single prompt with an arbitrary 50-iteration loop as the controller

**Answer:** `C`

**Explanation:** Anthropic advises using the simplest solution that works and only adding agentic autonomy when the task genuinely requires dynamic decision-making. A fixed, predictable pipeline is a textbook workflow — cheaper, more consistent, and easier to debug than an autonomous agent. Choosing an agent or hub-and-spoke here is over-engineering; the arbitrary-iteration-cap loop is anti-pattern #2.

**Tags:** `Domain 1` `Foundational` `Workflows vs Agents`

### Q3. When does Anthropic recommend choosing an agent over a workflow?

- **A.** Whenever the task involves more than one tool call
- **B.** Only when latency does not matter at all
- **C.** Whenever the customer explicitly asks for "AI"
- **D.** When the number of required steps is unpredictable and the model must decide the path dynamically based on intermediate results

**Answer:** `D`

**Explanation:** Agents shine when the problem space is open-ended, the number of steps cannot be predicted in advance, and the model must adapt its plan to what it discovers. Multiple tool calls alone do not justify agency — a workflow can chain tools. The deciding factor is unpredictability of the control flow, not tooling count or marketing language.

**Tags:** `Domain 1` `Foundational` `Workflows vs Agents`

### Q4. What is the central design principle Anthropic emphasizes before reaching for agents or multi-agent systems?

- **A.** Always maximize the number of subagents for redundancy
- **B.** Find the simplest solution possible and only increase complexity (workflow, then agent, then multi-agent) when simpler approaches fall short
- **C.** Always start with a multi-agent system so you never have to refactor later
- **D.** Prefer the largest model for every subtask regardless of cost

**Answer:** `B`

**Explanation:** Anthropic's repeated guidance is to start simple and add complexity only when it demonstrably improves outcomes, because agentic systems trade latency and cost for capability. Defaulting to multi-agent or always using the largest model inflates cost and latency without justification. Simplicity-first is a recurring exam signal across D1.

**Tags:** `Domain 1` `Foundational` `Architecture Selection`

### Q5. In the S2 Code-Generation scenario, a developer needs Claude Code to refactor a function the same way every run, with no deviation. Which architecture best fits?

- **A.** An autonomous agent free to explore the whole repository
- **B.** A multi-agent research system
- **C.** A deterministic workflow (a scripted, fixed sequence such as a slash command), because the task is repeatable and benefits from consistency
- **D.** A prompt that asks Claude to "be very consistent"

**Answer:** `C`

**Explanation:** A repeatable, well-defined transformation maps to a workflow — in Claude Code this is often a slash command or a tightly scoped script that runs the same steps each time. Asking the model to "be consistent" is prompt-based enforcement (anti-pattern #3) and gives no guarantee. Autonomy and multi-agent add nondeterminism the task does not need.

**Tags:** `Domain 1` `Foundational` `S2` `Architecture Selection`

## The Five Workflow Patterns — Identify and Select

### Q6. Anthropic's "Building effective agents" names five workflow patterns. Which list correctly enumerates them?

- **A.** Caching, batching, streaming, embedding, routing
- **B.** Prompt chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer
- **C.** MapReduce, scatter-gather, saga, circuit-breaker, bulkhead
- **D.** Plan mode, edit mode, review mode, test mode, deploy mode

**Answer:** `B`

**Explanation:** The five canonical workflow patterns are prompt chaining, routing, parallelization, orchestrator-workers, and evaluator-optimizer. The other options list API features, generic distributed-systems patterns, and Claude Code modes respectively — none are the five workflow patterns. Knowing this list verbatim is high-yield.

**Tags:** `Domain 1` `Foundational` `Workflow Patterns`

### Q7. A task decomposes into a fixed sequence where each step's output feeds the next, and you want to gate quality between steps. Which workflow pattern is this?

- **A.** Routing
- **B.** Parallelization
- **C.** Prompt chaining
- **D.** Evaluator-optimizer

**Answer:** `C`

**Explanation:** Prompt chaining breaks a task into a fixed sequence of LLM calls where each step processes the prior output, often with programmatic "gate" checks between steps. Routing classifies an input to one of several handlers — it does not chain sequential transformations. The defining cue here is "fixed sequence, output feeds next."

**Tags:** `Domain 1` `Foundational` `Workflow Patterns`

### Q8. Incoming support tickets fall into distinct categories (billing, technical, account) that each need different handling and prompts. Which pattern fits best?

- **A.** Prompt chaining
- **B.** Orchestrator-workers
- **C.** Evaluator-optimizer
- **D.** Routing

**Answer:** `D`

**Explanation:** Routing classifies an input and directs it to a specialized downstream prompt or handler, which is ideal when categories are distinct and benefit from separation of concerns. Prompt chaining is for sequential sub-steps, not classification-and-dispatch. In S1, routing a ticket to the right specialized handler is a classic routing application.

**Tags:** `Domain 1` `Intermediate` `S1` `Workflow Patterns`

### Q9. A document must be checked simultaneously against three independent rubrics (legal, brand, accuracy), and the results combined. Which workflow pattern applies?

- **A.** Parallelization (sectioning)
- **B.** Prompt chaining
- **C.** Routing
- **D.** Evaluator-optimizer

**Answer:** `A`

**Explanation:** Parallelization runs independent subtasks concurrently and aggregates them; the "sectioning" variant splits a task into independent slices (here, three rubrics) handled in parallel. Prompt chaining would serialize these and waste latency. The cue "simultaneously / independent / combine results" signals parallelization.

**Tags:** `Domain 1` `Intermediate` `Workflow Patterns`

### Q10. You ask Claude the same question several times in parallel and take a majority vote to improve reliability. Which workflow pattern (and variant) is this?

- **A.** Routing
- **B.** Orchestrator-workers
- **C.** Parallelization — the voting variant
- **D.** Evaluator-optimizer

**Answer:** `C`

**Explanation:** Parallelization has two variants: sectioning (independent subtasks) and voting (run the same task multiple times to gather diverse outputs and aggregate, e.g., by majority). Voting reduces variance on a single judgment. Orchestrator-workers dynamically decomposes a task; it is not the same as repeating one task for consensus.

**Tags:** `Domain 1` `Intermediate` `Workflow Patterns`

### Q11. A generator produces a draft and a separate evaluator critiques it, looping until the critique passes. Which pattern is this?

- **A.** Prompt chaining
- **B.** Parallelization (voting)
- **C.** Routing
- **D.** Evaluator-optimizer

**Answer:** `D`

**Explanation:** The evaluator-optimizer pattern pairs a generator with an evaluator that provides feedback in a loop until quality criteria are met. It differs from prompt chaining because the loop is feedback-driven and may repeat, not a fixed linear sequence. This pattern underpins multi-pass review in S5.

**Tags:** `Domain 1` `Intermediate` `S5` `Workflow Patterns`

### Q12. What is the key precondition for the evaluator-optimizer pattern to be worth its extra cost?

- **A.** The task must always finish in exactly one iteration
- **B.** You can articulate clear evaluation criteria, and iterative feedback measurably improves the output
- **C.** The evaluator must reuse the generator's exact prompt and context
- **D.** The generator must have at least 18 tools available

**Answer:** `B`

**Explanation:** Evaluator-optimizer pays off when criteria are expressible and feedback yields measurable gains — like a human reviewer whose comments improve a draft. Reusing the generator's exact prompt/context as the evaluator is same-session self-review (anti-pattern #9), which inherits the author's bias. Tool count is irrelevant to this pattern.

**Tags:** `Domain 1` `Intermediate` `S5` `Workflow Patterns`

### Q13. In S6 (Structured Data Extraction), the system extracts data, validates it against a JSON Schema, and re-prompts on validation failure until it passes. Which workflow pattern most closely describes this loop?

- **A.** Routing
- **B.** Evaluator-optimizer (with the schema validator acting as the deterministic evaluator)
- **C.** Parallelization (sectioning)
- **D.** Orchestrator-workers

**Answer:** `B`

**Explanation:** A generate→validate→re-prompt loop is the evaluator-optimizer pattern where the "evaluator" is a deterministic schema validator rather than another LLM. Using code as the evaluator is actually stronger than an LLM critic for objective constraints. Routing and sectioning do not describe a corrective feedback loop.

**Tags:** `Domain 1` `Intermediate` `S6` `Workflow Patterns`

### Q14. Which statement about choosing among the five workflow patterns is correct?

- **A.** Always use orchestrator-workers because it subsumes the others
- **B.** Pick the simplest pattern that matches the task's structure; combine patterns only when the structure genuinely calls for it
- **C.** Routing is forbidden once you have more than two categories
- **D.** Parallelization should never be combined with prompt chaining

**Answer:** `B`

**Explanation:** Patterns are building blocks chosen by task structure, and they compose (e.g., a router that dispatches to a chain). Defaulting to the heaviest pattern over-engineers. There is no rule forbidding many routing categories or combining parallelization with chaining; composition is encouraged when warranted.

**Tags:** `Domain 1` `Intermediate` `Workflow Patterns`

### Q15. A workflow needs to break a 200-page report into per-chapter summaries generated concurrently, then a final pass to merge them. Which combination is most appropriate?

- **A.** Pure routing with no aggregation
- **B.** Parallelization (sectioning) for per-chapter summaries, followed by a chaining/aggregation step to merge
- **C.** Evaluator-optimizer with a single generator and no parallel work
- **D.** A single agent looping until it says "I'm done"

**Answer:** `B`

**Explanation:** Independent per-chapter work is sectioning (parallelization), and the final merge is a downstream aggregation step — a clean composition of two patterns. The single agent ending on "I'm done" is anti-pattern #1 (parsing natural language for termination). Pure routing lacks the needed aggregation.

**Tags:** `Domain 1` `Advanced` `Workflow Patterns`

## Orchestrator-Worker and Hub-and-Spoke Responsibilities

### Q16. In the orchestrator-workers pattern, what is the orchestrator's primary responsibility?

- **A.** To execute every tool call itself and never delegate
- **B.** To dynamically break the task into subtasks, delegate them to worker LLMs, and synthesize their results
- **C.** To enforce a hard 5-iteration cap as the main control mechanism
- **D.** To parse worker text for the phrase "task complete"

**Answer:** `B`

**Explanation:** The orchestrator decomposes the problem at runtime, dispatches subtasks to workers, and integrates their outputs — the decomposition is dynamic, which distinguishes it from static parallelization. Executing everything itself defeats the pattern. The iteration-cap and prose-parsing options describe anti-patterns #2 and #1 respectively, not orchestrator duties.

**Tags:** `Domain 1` `Intermediate` `S3` `Orchestrator-Workers`

### Q17. How does orchestrator-workers differ from the parallelization pattern?

- **A.** Orchestrator-workers decomposes subtasks dynamically at runtime, whereas parallelization splits a predefined set of subtasks known in advance
- **B.** They are identical; the names are interchangeable
- **C.** Parallelization can only use one worker
- **D.** Orchestrator-workers cannot aggregate results

**Answer:** `A`

**Explanation:** The crucial difference is dynamism: in parallelization the subtasks are fixed and known ahead of time, while in orchestrator-workers the orchestrator determines the subtasks based on the specific input. This is why orchestrator-workers suits open-ended research (S3) where you cannot predict the needed subtasks. The other options are factually wrong.

**Tags:** `Domain 1` `Advanced` `S3` `Orchestrator-Workers`

### Q18. In the S3 multi-agent research system using a hub-and-spoke (coordinator/subagent) design, what should the coordinator own?

- **A.** The detailed low-level web searches for every subtopic
- **B.** High-level planning, subtask delegation, and final synthesis — while subagents do the focused information-gathering
- **C.** Nothing; subagents should self-organize with no coordinator
- **D.** Parsing each subagent's prose to guess when research is finished

**Answer:** `B`

**Explanation:** In hub-and-spoke, the coordinator (hub) plans, spawns and delegates to subagents (spokes), then synthesizes; subagents handle focused gathering within isolated contexts. Pushing all low-level searches onto the coordinator overloads its context and defeats isolation. Guessing completion from prose is the natural-language-termination anti-pattern (#1).

**Tags:** `Domain 1` `Intermediate` `S3` `Hub-and-Spoke`

### Q19. Anthropic's multi-agent research write-up reports a major benefit of giving each subagent its own context window. What is it?

- **A.** It eliminates all token cost
- **B.** It guarantees the subagents never make mistakes
- **C.** It lets each subagent explore a slice of the problem in parallel without polluting the others' context, expanding the system's effective working memory
- **D.** It removes the need for any coordinator

**Answer:** `C`

**Explanation:** Separate context windows let subagents pursue independent lines of inquiry in parallel and keep their intermediate clutter out of each other's context, effectively scaling total working memory and enabling breadth. It does not eliminate cost — multi-agent uses more tokens — nor remove the coordinator or guarantee correctness.

**Tags:** `Domain 1` `Advanced` `S3` `Subagent Isolation`

### Q20. Anthropic notes that multi-agent systems consume substantially more tokens than single-agent chat. What follows for architecture decisions?

- **A.** Never use multi-agent under any circumstances
- **B.** Reserve multi-agent designs for high-value tasks where breadth/parallelism justifies the higher token cost and latency
- **C.** Multi-agent is always the cheapest option
- **D.** Token cost is irrelevant once you add subagents

**Answer:** `B`

**Explanation:** Because subagents multiply token usage, multi-agent architectures are justified mainly for valuable, parallelizable tasks (like broad research) where the gains outweigh cost. It is not banned outright; it is simply not free. The exam tests whether you weigh cost/latency against capability before going multi-agent.

**Tags:** `Domain 1` `Advanced` `S3` `Cost/Latency Trade-offs`

### Q21. A coordinator in S3 spawns five subagents but passes each a vague instruction like "research the topic." What is the most likely failure and the fix?

- **A.** No failure; vague prompts are fine for subagents
- **B.** Subagents duplicate work and leave gaps; fix by having the coordinator give each a clear, distinct objective, scope, and output format
- **C.** The fix is to add a 50-iteration cap to each subagent
- **D.** The fix is to let subagents read each other's context windows freely

**Answer:** `B`

**Explanation:** Anthropic found that vague delegation causes subagents to overlap or miss areas; effective coordinators give detailed, differentiated task descriptions (objective, scope, expected output, tools). Adding an iteration cap does not fix poor delegation. Letting subagents share contexts defeats the isolation that makes the pattern scale.

**Tags:** `Domain 1` `Advanced` `S3` `Hub-and-Spoke`

### Q22. Which responsibility should NOT belong to a worker/subagent in a well-designed hub-and-spoke system?

- **A.** Performing its assigned focused subtask with its curated tools
- **B.** Deciding the overall mission strategy and re-planning the whole project for the other subagents
- **C.** Returning a structured result to the coordinator
- **D.** Operating within its own isolated context window

**Answer:** `B`

**Explanation:** Strategy and cross-subagent re-planning are the coordinator's job; a worker that tries to re-plan the entire mission breaks the separation of concerns and creates conflicting plans. Performing its subtask, returning structured results, and running in isolation are all proper worker duties.

**Tags:** `Domain 1` `Intermediate` `S3` `Hub-and-Spoke`

### Q23. When is a single-agent (no subagents) design preferable to hub-and-spoke for a research-style task?

- **A.** Always; single agents are universally better
- **B.** Only when you have exactly 18 tools
- **C.** When the task is narrow, mostly sequential, and does not benefit from parallel breadth — avoiding multi-agent token/latency overhead
- **D.** Never; research always needs subagents

**Answer:** `C`

**Explanation:** Multi-agent overhead is only worth it when parallel breadth helps; a narrow, sequential task is handled more cheaply and simply by one agent. Neither "always" nor "never" is correct — it depends on whether the task parallelizes. Tool count is unrelated to this decision.

**Tags:** `Domain 1` `Advanced` `S3` `Architecture Selection`

## The Agent Loop and stop_reason-Based Termination

### Q24. In the Claude Messages API agentic loop, what signals that the model wants to call a tool and the loop should execute it?

- **A.** The response text contains the word "tool"
- **B.** The response has stop_reason "tool_use" and includes one or more tool_use content blocks
- **C.** The model returns an HTTP 429
- **D.** The usage.output_tokens field exceeds a threshold

**Answer:** `B`

**Explanation:** When Claude decides to use a tool, the API returns stop_reason `tool_use` along with tool_use blocks specifying the tool name and input; your loop runs the tool and returns a tool_result. Scanning text for "tool" is brittle natural-language parsing. A 429 is rate-limiting, and token counts do not indicate tool intent.

**Tags:** `Domain 1` `Foundational` `Agent Loop`

### Q25. After your code executes a tool, how do you continue the agent loop correctly?

- **A.** Append the assistant's tool_use turn and a user turn containing the corresponding tool_result block(s), then call the API again
- **B.** Start a brand-new conversation with no history
- **C.** Parse the tool output and ask the user to paste it back
- **D.** Immediately stop, because one tool call is always enough

**Answer:** `A`

**Explanation:** The loop continues by sending the model its own tool_use turn plus a user message carrying the tool_result(s) keyed by tool_use_id, so Claude can incorporate the output and decide the next step. Wiping history loses context. Tool results must be returned to the model programmatically, not via the user, and a single call rarely completes multi-step tasks.

**Tags:** `Domain 1` `Intermediate` `Agent Loop`

### Q26. Which stop_reason indicates the model has finished its turn naturally with no further tool call requested?

- **A.** tool_use
- **B.** end_turn
- **C.** max_tokens
- **D.** stop_sequence

**Answer:** `B`

**Explanation:** `end_turn` means Claude completed its response on its own and is not requesting a tool, so the agent loop can terminate for that turn. `tool_use` means continue by running a tool; `max_tokens` means the output was truncated by the limit; `stop_sequence` means a configured stop string was hit. Reading stop_reason is the correct, deterministic control signal.

**Tags:** `Domain 1` `Intermediate` `Agent Loop`

### Q27. An engineer terminates the agent loop when the assistant text contains "I'm done" or "task complete." Why is this an anti-pattern?

- **A.** Because those phrases are too short to match
- **B.** Because natural-language phrases are unreliable termination signals; the loop should branch on the API stop_reason instead
- **C.** Because the model never produces those phrases
- **D.** Because it terminates too slowly

**Answer:** `B`

**Explanation:** This is anti-pattern #1: parsing natural language for loop control is brittle — the model may say "done" mid-task or never say it, and phrasing varies. The deterministic signal is stop_reason (e.g., `end_turn` vs `tool_use`). The flaw is reliability of the signal, not its length or speed.

**Tags:** `Domain 1` `Intermediate` `S3` `Termination Anti-Pattern`

### Q28. What is the correct role of a maximum-iteration cap in an agent loop?

- **A.** It is the primary mechanism that decides when the task is complete
- **B.** It is a safety backstop against runaway loops, not the primary completion signal — completion should be driven by stop_reason and task state
- **C.** It should be removed entirely because stop_reason handles everything
- **D.** It must always be set to exactly 50

**Answer:** `B`

**Explanation:** An iteration cap is a guardrail to prevent infinite loops and bound cost; relying on it as the main stopping criterion is anti-pattern #2. Real completion is determined by stop_reason and whether the task goal is met. The cap is still useful as a backstop (so not removed), and there is no universal magic number.

**Tags:** `Domain 1` `Intermediate` `Termination Anti-Pattern`

### Q29. In S3, a research subagent should stop when it has gathered sufficient evidence for its assigned subtask. What is the most robust completion design?

- **A.** Stop after a fixed 10 tool calls regardless of findings
- **B.** Let the model decide it is finished (returning end_turn after producing its structured result) within a sane max-iteration backstop, then have the coordinator verify completeness
- **C.** Stop as soon as the subagent's text says "I think that's enough"
- **D.** Never stop; keep searching until the context window overflows

**Answer:** `B`

**Explanation:** Robust termination combines the model's own end_turn signal (after delivering the required structured output) with a backstop cap and a coordinator-level completeness check. A hard fixed count is anti-pattern #2; the prose-based stop is anti-pattern #1; running until overflow is reckless and corrupts context.

**Tags:** `Domain 1` `Advanced` `S3` `Termination`

### Q30. Which stop_reason should trigger your loop to handle a truncated/incomplete response rather than treat the task as done?

- **A.** end_turn
- **B.** tool_use
- **C.** max_tokens
- **D.** pause_turn

**Answer:** `C`

**Explanation:** `max_tokens` means the model hit the output token limit mid-response, so the content is likely incomplete; you should handle it (e.g., continue generation or raise max_tokens) rather than assume success. `end_turn` is a clean finish, and `tool_use` means run a tool. Treating a truncated response as complete silently degrades reliability.

**Tags:** `Domain 1` `Intermediate` `Agent Loop` `Reliability`

### Q31. Why is branching on stop_reason more reliable than counting tool calls to decide when an agent is finished?

- **A.** Counting is computationally expensive
- **B.** stop_reason is a deterministic, model-emitted control signal tied to the model's actual decision, whereas a call count is an arbitrary proxy that does not reflect task completion
- **C.** Tool calls cannot be counted accurately
- **D.** stop_reason is only available in batch mode

**Answer:** `B`

**Explanation:** stop_reason directly reflects whether Claude wants to continue (tool_use) or has finished (end_turn), giving deterministic control aligned with the model's intent. A call count is an arbitrary heuristic that can stop too early or too late. stop_reason is available on normal Messages responses, not just batch.

**Tags:** `Domain 1` `Intermediate` `Agent Loop`

### Q32. An agent must run a tool, read its result, and then possibly run more tools. What is the minimal correct loop structure?

- **A.** Call API once, ignore stop_reason, return the first text block
- **B.** Loop: call API → if stop_reason is tool_use, execute tools, append results, repeat; if end_turn, stop (with a max-iteration backstop)
- **C.** Call the tool first, then call the API exactly once
- **D.** Ask the model in the system prompt to "please stop when done" and trust it

**Answer:** `B`

**Explanation:** The canonical agent loop calls the API, checks stop_reason, executes tools and feeds results back while stop_reason is tool_use, and exits on end_turn, bounded by a backstop. Ignoring stop_reason breaks multi-step tasks. Relying on a polite system-prompt instruction is prompt-based enforcement (anti-pattern #3), not a control mechanism.

**Tags:** `Domain 1` `Intermediate` `Agent Loop`

## Fault Tolerance — Backoff, State, and Error Handling

### Q33. A tool call in S1 fails with a transient HTTP 503 from an upstream MCP server. What is the correct fault-tolerance response?

- **A.** Silently return an empty result so the agent thinks it succeeded
- **B.** Retry with exponential backoff (and jitter), and if it still fails, return a descriptive error to the model so it can adapt
- **C.** Crash the entire agent process immediately
- **D.** Replace the error with a hard-coded fake success payload

**Answer:** `B`

**Explanation:** Transient failures call for retry with exponential backoff and jitter; persistent failures should surface a clear, diagnostic error to the model so it can route around the problem or escalate. Returning an empty result or fake success is anti-pattern #7 (silently suppressing errors), which corrupts downstream reasoning. Crashing abandons recoverable work.

**Tags:** `Domain 1` `Intermediate` `S1` `Fault Tolerance`

### Q34. Why does Anthropic recommend exponential backoff (with jitter) over fixed-interval immediate retries for rate limits and transient errors?

- **A.** Fixed intervals are illegal in the API
- **B.** Exponential backoff reduces load on a struggling service and avoids synchronized retry storms; jitter de-correlates concurrent clients
- **C.** Backoff guarantees the request will never fail again
- **D.** Immediate retries are always faster end-to-end

**Answer:** `B`

**Explanation:** Backoff progressively spaces retries so a recovering service is not hammered, and jitter prevents many clients from retrying in lockstep (the thundering-herd problem). It does not guarantee success. Immediate tight retries can worsen congestion and trigger more 429s, slowing overall completion.

**Tags:** `Domain 1` `Intermediate` `Fault Tolerance`

### Q35. To make a long-running agent fault-tolerant against process restarts, what should the architecture persist?

- **A.** Only the final answer, discarding all intermediate state
- **B.** Only the system prompt
- **C.** Durable checkpoints of conversation/tool state so the agent can resume from the last good point instead of restarting from scratch
- **D.** Nothing; agents should always restart cold

**Answer:** `C`

**Explanation:** Persisting conversation history, tool results, and progress as durable state lets the agent resume after a crash or restart without redoing expensive work. Saving only the final answer or system prompt provides nothing to resume from, and cold restarts waste tokens, time, and money on long tasks.

**Tags:** `Domain 1` `Advanced` `Fault Tolerance` `State`

### Q36. A tool encounters a real error but the wrapper returns a generic message: "Error." Why is this poor agent design?

- **A.** It is fine; the model does not need details
- **B.** It hides diagnostic context the model needs to recover (anti-pattern #6); error results should be specific and actionable so the model can adjust its approach
- **C.** Generic messages are too long
- **D.** The model cannot read tool_result blocks at all

**Answer:** `B`

**Explanation:** Generic errors strip away the information (which field, which constraint, which resource) the model would use to self-correct — this is anti-pattern #6. Returning a specific, structured error (e.g., "lookup failed: customer_id not found for ID 123") lets Claude retry intelligently or escalate. The model absolutely can read tool_result content.

**Tags:** `Domain 1` `Intermediate` `S1` `Error Handling`

### Q37. In S1, a payment-refund tool throws because the customer record is missing. Which tool_result is best for agent recovery?

- **A.** Return {} as if it succeeded
- **B.** Return a structured error: success=false, error_code="customer_not_found", message="No customer with id=... ; ask the user to confirm their account email"
- **C.** Throw and kill the agent
- **D.** Return "Something went wrong" with no fields

**Answer:** `B`

**Explanation:** A structured, descriptive error gives the model an actionable path (ask the user to confirm identity) and preserves the failure semantics. Returning {} is silent suppression (anti-pattern #7); the bare generic message is anti-pattern #6; killing the agent abandons a recoverable conversation. Good errors are part of fault tolerance.

**Tags:** `Domain 1` `Intermediate` `S1` `Error Handling`

### Q38. Which combination best describes a fault-tolerant agentic tool layer?

- **A.** Retries with backoff, idempotent operations where possible, descriptive structured errors, and durable state for resumption
- **B.** No retries, hidden errors, and cold restarts
- **C.** Infinite immediate retries with no cap
- **D.** Returning fake success to keep the loop moving

**Answer:** `A`

**Explanation:** A robust tool layer retries transient failures with backoff, prefers idempotent operations so retries are safe, surfaces descriptive errors for model recovery, and checkpoints state for resumption. Hidden errors and fake success are anti-patterns #6/#7; uncapped immediate retries cause retry storms and unbounded cost.

**Tags:** `Domain 1` `Advanced` `Fault Tolerance`

### Q39. Why does idempotency matter when an agent retries a tool after a timeout?

- **A.** It does not matter; retries are always safe
- **B.** Because the original call may have succeeded server-side before the timeout; idempotency prevents duplicate side effects (e.g., double refunds) on retry
- **C.** Idempotency makes responses shorter
- **D.** Only read tools need to be idempotent

**Answer:** `B`

**Explanation:** A timeout does not guarantee the operation did not run; without idempotency (e.g., an idempotency key), retrying a write like a refund can apply it twice. This is precisely why mutating tools in S1 need idempotency guarantees. Read tools are naturally safe, but writes are the risky case, and retries are not automatically safe.

**Tags:** `Domain 1` `Advanced` `S1` `Fault Tolerance`

### Q40. A subagent in S3 fails halfway through its assignment. What is the most resilient coordinator behavior?

- **A.** Abort the entire research mission
- **B.** Silently drop the failed subagent and report 100% completeness
- **C.** Detect the failure via a structured error, retry the subtask (possibly with adjusted scope), and continue with the other subagents' results — degrading gracefully
- **D.** Loop the failed subagent forever with no backstop

**Answer:** `C`

**Explanation:** Resilient coordinators isolate failure to one spoke: they catch the structured error, retry or re-scope, and still synthesize from the successful subagents while flagging gaps. Reporting full completeness after a silent drop is anti-pattern #7 (and masks a failed slice). Aborting everything is brittle; an unbounded retry loop violates the backstop principle.

**Tags:** `Domain 1` `Advanced` `S3` `Fault Tolerance`

## Subagent Isolation and Context Boundaries

### Q41. What is the primary architectural reason to give a subagent its own isolated context window rather than sharing the coordinator's?

- **A.** To make the subagent slower
- **B.** To prevent context pollution and token bloat, so each subagent reasons over only its relevant slice and the coordinator's context stays clean
- **C.** Because subagents cannot share memory technically
- **D.** To avoid using stop_reason

**Answer:** `B`

**Explanation:** Isolation keeps each subagent focused on its subtask without dragging in unrelated tokens, and it protects the coordinator's context from clutter — improving both reasoning quality and token efficiency. It is an intentional design choice, not a technical impossibility, and it has nothing to do with stop_reason or being slower.

**Tags:** `Domain 1` `Intermediate` `S3` `Subagent Isolation`

### Q42. In Claude Code, when you delegate work to a subagent (e.g., via the Task tool), what is the key context-management benefit?

- **A.** The subagent's intermediate exploration is kept out of the main agent's context, returning only a concise result
- **B.** The subagent shares every token with the main thread in real time
- **C.** The subagent automatically has access to all 18 tools at once
- **D.** The main agent loses its conversation history

**Answer:** `A`

**Explanation:** Subagent delegation isolates the noisy intermediate steps (file reads, searches, dead ends) in a separate context and returns just the distilled result to the parent, conserving the main context window. The main thread keeps its history, tokens are not streamed back wholesale, and tool count is curated, not maxed out (which would be anti-pattern #8).

**Tags:** `Domain 1` `Intermediate` `S4` `Subagent Isolation`

### Q43. A subagent returns a 30,000-token raw dump of everything it explored. What should it return instead, and why?

- **A.** The full dump, so the coordinator has maximum information
- **B.** A concise, structured summary of relevant findings, to keep the coordinator's context clean and focused
- **C.** Nothing, to save tokens
- **D.** A natural-language note saying "I'm done" with no findings

**Answer:** `B`

**Explanation:** The value of subagent isolation is that workers digest their exploration and hand back a compact, structured result — dumping raw context defeats the purpose and bloats the coordinator. Returning nothing loses the work; returning a bare "done" is anti-pattern #1 and provides no usable output.

**Tags:** `Domain 1` `Intermediate` `S3` `Subagent Isolation`

### Q44. Which scenario best justifies spawning multiple isolated subagents rather than a single agent?

- **A.** A short FAQ lookup
- **B.** A broad research question requiring parallel investigation of many independent sources, where each subagent explores one branch
- **C.** Reformatting a single JSON object
- **D.** Echoing the user's message back

**Answer:** `B`

**Explanation:** Multiple isolated subagents pay off when the task has independent branches that benefit from parallel breadth and separate contexts — Anthropic's research system is the canonical example. Trivial tasks gain nothing from subagents and incur extra token/latency cost, violating the simplicity-first principle.

**Tags:** `Domain 1` `Intermediate` `S3` `Architecture Selection`

### Q45. What risk arises if subagents are allowed to write to a shared mutable scratchpad without coordination?

- **A.** None; shared mutable state is always fine
- **B.** The subagents will run faster
- **C.** Race conditions and inconsistent state, undermining the isolation that makes the pattern reliable; coordination or append-only/partitioned writes are needed
- **D.** stop_reason stops working

**Answer:** `C`

**Explanation:** Uncoordinated concurrent writes create races and corrupt shared state, eroding the predictability isolation was meant to provide. The fix is to partition writes, use append-only logs, or have the coordinator own the merge. Shared mutable state is not "always fine"; concurrency bugs do not speed things up and are unrelated to stop_reason.

**Tags:** `Domain 1` `Advanced` `S3` `Subagent Isolation`

## Human-Escalation Logic — Complexity, Not Sentiment or Self-Confidence

### Q46. In S1, when should the support agent escalate a ticket to a human, per best practice?

- **A.** Whenever the customer's sentiment is negative or angry
- **B.** When the task exceeds the agent's reliable capability or authority — e.g., requires actions/judgment outside its tools, policy thresholds, or verified scope
- **C.** Whenever the model self-reports low confidence in a numeric score
- **D.** After a fixed number of messages regardless of progress

**Answer:** `B`

**Explanation:** Escalation should be driven by task complexity and the boundaries of the agent's competence/authority (e.g., a refund above its limit, a policy exception, missing tools). Sentiment-based escalation is anti-pattern #5 — an angry customer with a simple request does not need a human. Self-reported confidence is anti-pattern #4; fixed message counts are an arbitrary proxy.

**Tags:** `Domain 1` `Intermediate` `S1` `Escalation`

### Q47. Why is using the model's self-reported confidence score as the primary escalation trigger an anti-pattern?

- **A.** Confidence scores are always exactly correct
- **B.** Self-reported confidence is poorly calibrated and gameable; escalation should be based on objective complexity/authority signals, not the model's own number
- **C.** The API forbids returning confidence
- **D.** Confidence scores cost too many tokens

**Answer:** `B`

**Explanation:** This is anti-pattern #4: a model's stated confidence is not reliably calibrated to actual correctness and can be high on wrong answers, so it is a weak control signal. Better triggers are deterministic: action exceeds authorized limits, required tool unavailable, policy flag matched. The issue is calibration, not API rules or token cost.

**Tags:** `Domain 1` `Intermediate` `S1` `Escalation Anti-Pattern`

### Q48. A frustrated but straightforward customer asks to reset a password. The agent escalates because sentiment is negative. What went wrong?

- **A.** Nothing; negative sentiment always warrants a human
- **B.** Sentiment was conflated with complexity (anti-pattern #5); a simple, in-scope task should be handled by the agent regardless of tone
- **C.** The agent should have used a 50-iteration cap
- **D.** The agent should have parsed "I'm furious" for termination

**Answer:** `B`

**Explanation:** Sentiment and task complexity are orthogonal — a simple password reset is fully within the agent's competence even if the customer is upset. Escalating on tone (anti-pattern #5) wastes human capacity and degrades resolution time. The fix is to escalate on complexity/authority, not emotion. The cap and termination options are unrelated anti-patterns.

**Tags:** `Domain 1` `Foundational` `S1` `Escalation Anti-Pattern`

### Q49. Which escalation rule is best implemented as a deterministic hook rather than left to the prompt?

- **A.** "Escalate any refund over $500 to a human approver"
- **B.** "Try to be helpful and escalate if it feels hard"
- **C.** "Escalate when the model says it is unsure"
- **D.** "Escalate when the customer seems annoyed"

**Answer:** `A`

**Explanation:** A hard policy threshold like a refund ceiling is a critical business rule that must be enforced deterministically in code/hooks, not by asking the model nicely (which is anti-pattern #3). Vague prompt instructions, self-confidence (anti-pattern #4), and sentiment (anti-pattern #5) are all unreliable triggers for a binding policy.

**Tags:** `Domain 1` `Intermediate` `S1` `Escalation`

### Q50. What is the cleanest signal that a multi-step task should hand off to a human, in an Agent SDK support flow?

- **A.** The conversation reached 12 turns
- **B.** The task requires an action the agent is not authorized/equipped to perform, or hits a defined policy boundary detected in code
- **C.** The customer used an exclamation point
- **D.** The model emitted the token "human"

**Answer:** `B`

**Explanation:** Authority/capability boundaries and policy matches are objective, code-detectable conditions that make escalation deterministic and auditable. Turn counts and punctuation/sentiment are arbitrary proxies, and trusting the model to emit a magic token is brittle natural-language control. Escalate on what the agent can and may do, not on heuristics.

**Tags:** `Domain 1` `Intermediate` `S1` `Escalation`

### Q51. The most defensible escalation architecture combines which elements?

- **A.** Sentiment classifier plus self-reported confidence
- **B.** Deterministic policy checks (limits, authorization, required-but-missing tools) plus complexity heuristics, with the human handoff itself enforced in code
- **C.** A prompt that says "escalate if needed" and nothing else
- **D.** An iteration cap that escalates at exactly turn 50

**Answer:** `B`

**Explanation:** Robust escalation layers objective policy/authorization checks with complexity signals and enforces the handoff deterministically so critical rules cannot be skipped by the model. Sentiment plus confidence stacks two anti-patterns (#5 and #4). A bare prompt instruction is anti-pattern #3; a turn-50 cap is the iteration-cap anti-pattern #2.

**Tags:** `Domain 1` `Advanced` `S1` `Escalation`

## Deterministic Enforcement vs. Prompt-Based Control

### Q52. A critical compliance rule must never be violated by the agent. Where should it be enforced?

- **A.** Only by instructing the model in the system prompt to always follow it
- **B.** In deterministic code/hooks/guardrails that the model cannot bypass, with the prompt as a secondary aid
- **C.** By asking the model to rate its own compliance
- **D.** By raising the temperature so the model is more careful

**Answer:** `B`

**Explanation:** Critical business/compliance rules require deterministic enforcement (code, hooks, validators) because prompts are probabilistic and can be ignored or jailbroken — relying on the prompt alone is anti-pattern #3. The prompt can reinforce intent, but the guarantee lives in code. Self-rating is anti-pattern #4; temperature does not enforce rules.

**Tags:** `Domain 1` `Intermediate` `Deterministic Enforcement`

### Q53. In Claude Code, which mechanism deterministically blocks an action (e.g., editing protected files) regardless of what the model decides?

- **A.** A polite line in CLAUDE.md asking it not to
- **B.** A pre-tool-use hook that inspects the proposed action and denies it
- **C.** A higher max_tokens setting
- **D.** A larger system prompt

**Answer:** `B`

**Explanation:** Hooks in Claude Code run deterministic code around tool calls and can block or modify actions independent of the model's reasoning — the right place for hard constraints. A request in CLAUDE.md is guidance, not enforcement (anti-pattern #3 if used as the sole guarantee). Token limits and prompt size do not gate actions.

**Tags:** `Domain 1` `Intermediate` `S2` `Deterministic Enforcement`

### Q54. Why is "deterministic guardrail in code" preferred over "strong wording in the prompt" for safety-critical constraints?

- **A.** Prompts are more expensive than code
- **B.** Model behavior is probabilistic and can deviate; code-level guardrails provide a hard, auditable guarantee that does not depend on sampling
- **C.** Code is always shorter than a prompt
- **D.** Prompts cannot mention constraints at all

**Answer:** `B`

**Explanation:** The core reason is determinism: a guardrail in code enforces the rule every time and is auditable, whereas prompt instructions are followed only probabilistically. This is the essence of anti-pattern #3. Cost and length are irrelevant, and prompts certainly can mention constraints — they just cannot guarantee them.

**Tags:** `Domain 1` `Intermediate` `Deterministic Enforcement`

## Cost and Latency Trade-offs

### Q55. Which statement best captures the cost/latency trade-off of agentic systems?

- **A.** Agents are always cheaper and faster than workflows
- **B.** Agentic autonomy and multi-agent breadth increase capability but cost more tokens and add latency, so the gain must justify the spend
- **C.** Latency is irrelevant in production
- **D.** Token cost only matters for embeddings

**Answer:** `B`

**Explanation:** Anthropic frames agents and especially multi-agent systems as trading higher token cost and latency for greater capability and breadth; you should adopt them only when that trade is worth it. Agents are not universally cheaper, latency matters for UX, and token cost applies broadly, not just to embeddings.

**Tags:** `Domain 1` `Foundational` `Cost/Latency Trade-offs`

### Q56. To reduce latency in a parallelizable D1 workflow, which technique is most directly applicable?

- **A.** Run independent subtasks concurrently (parallelization) instead of serially
- **B.** Increase the iteration cap to 100
- **C.** Add 18 tools to one agent
- **D.** Disable stop_reason checks

**Answer:** `A`

**Explanation:** Parallelization runs independent subtasks at the same time, cutting wall-clock latency for tasks that decompose into independent slices. Raising the iteration cap does nothing for latency and risks runaway loops; cramming tools into one agent is anti-pattern #8; disabling stop_reason breaks correct termination.

**Tags:** `Domain 1` `Intermediate` `Cost/Latency Trade-offs`

### Q57. A high-volume, non-interactive batch of independent classification jobs needs the lowest cost. Which approach fits S5/S6 best?

- **A.** Submit them one at a time to the synchronous Messages API at high concurrency
- **B.** Use the Message Batches API for asynchronous, lower-cost bulk processing of independent requests
- **C.** Build a multi-agent hub-and-spoke per job
- **D.** Use extended thinking with maximum budget on every tiny job

**Answer:** `B`

**Explanation:** The Batches API is designed for large volumes of independent, non-time-sensitive requests at reduced cost — ideal for offline classification in CI/CD (S5) or bulk extraction (S6). Per-request synchronous calls cost more and add orchestration burden. A hub-and-spoke per job and max extended thinking on trivial jobs are gross over-engineering.

**Tags:** `Domain 1` `Intermediate` `S5` `Cost/Latency Trade-offs`

### Q58. When is it appropriate to route simpler subtasks to a smaller/cheaper model while reserving a larger model for hard steps?

- **A.** Never; always use the largest model everywhere
- **B.** When subtasks vary in difficulty — a routing/orchestration design can send easy steps to a cheaper model and only escalate hard steps, optimizing cost without sacrificing quality where it matters
- **C.** Only in batch mode
- **D.** Only if you have exactly four tools

**Answer:** `B`

**Explanation:** Model-tiering by subtask difficulty (often via routing or an orchestrator) is a standard cost optimization: cheap models handle the easy bulk while a stronger model handles the genuinely hard steps. Always using the biggest model wastes money on trivial work. This is not restricted to batch or tied to tool counts.

**Tags:** `Domain 1` `Advanced` `Cost/Latency Trade-offs`

### Q59. Prompt caching is most beneficial for reducing cost/latency in which agentic pattern?

- **A.** A workflow with a large, stable system prompt/context reused across many calls (e.g., the same tools+instructions in every agent-loop turn)
- **B.** A one-off single call that never repeats any context
- **C.** A request where every token of input changes each call
- **D.** Embedding generation only

**Answer:** `A`

**Explanation:** Prompt caching pays off when a large prefix (system prompt, tool definitions, reference docs) is reused across many calls — exactly the case in an agent loop or a high-throughput workflow, cutting cost and time-to-first-token on cache hits. A single non-repeating call or fully-variable input gets no cache reuse, and caching is not specific to embeddings.

**Tags:** `Domain 1` `Intermediate` `Cost/Latency Trade-offs`

### Q60. Adding subagents to an S3 research system improved breadth but tripled cost and doubled latency for a low-value internal query. What is the right call?

- **A.** Keep the multi-agent design because more agents are always better
- **B.** Downgrade to a single-agent (or simpler workflow) design for this low-value query, since the cost/latency no longer justifies the breadth
- **C.** Add even more subagents to amortize cost
- **D.** Remove stop_reason handling to save tokens

**Answer:** `B`

**Explanation:** When the value of the task does not justify the multi-agent overhead, the correct response is to simplify back to a single agent or workflow — match architecture complexity to task value. "More agents are always better" and "add more to amortize" ignore the cost/latency reality. Removing stop_reason breaks correctness and saves little.

**Tags:** `Domain 1` `Advanced` `S3` `Cost/Latency Trade-offs`

## Mixed Scenario Application — S1 / S3 Emphasis

### Q61. In S1, the support agent has 18 tools spanning billing, technical, account, and reporting domains, and it frequently picks the wrong one. What is the best architectural fix?

- **A.** Add more tools so it always has an option
- **B.** Curate a tight tool set (~4–7) per agent and split the domains across specialized subagents or MCP boundaries
- **C.** Tell the model in the prompt to "choose tools carefully"
- **D.** Increase max_tokens so it can reason longer

**Answer:** `B`

**Explanation:** Eighteen tools is anti-pattern #8 — too many choices degrade tool-selection accuracy. The fix is a curated set per agent and decomposition by domain into subagents or MCP servers so each agent reasons over a focused toolset. Adding tools worsens it; prompt pleading is anti-pattern #3; more tokens does not address the selection problem.

**Tags:** `Domain 1` `Intermediate` `S1` `Tool Curation`

### Q62. An S1 agent silently returns an empty list when its knowledge-base search tool errors, and then confidently tells the customer "no results found." What two anti-patterns are present?

- **A.** #7 silent error suppression and #6 hiding diagnostic context from the model
- **B.** #1 natural-language termination and #2 iteration cap
- **C.** #4 self-confidence and #5 sentiment
- **D.** #9 self-review and #10 aggregate metrics

**Answer:** `A`

**Explanation:** Swallowing the error and returning an empty result as if successful is anti-pattern #7, and presenting no diagnostic detail to the model (so it cannot tell "no data" from "tool failed") is anti-pattern #6. The fix is a structured error so the agent can retry or escalate rather than fabricate a clean "no results." The other option pairs are unrelated to this failure.

**Tags:** `Domain 1` `Advanced` `S1` `Error Handling`

### Q63. In S3, the coordinator decides the research is complete by checking whether each subagent returned a valid structured result for its assigned subtask. Why is this better than reading the subagents' prose?

- **A.** It is not better; prose is more reliable
- **B.** Structured-result presence/validity is a deterministic completion signal aligned with the assigned contract, whereas prose parsing is brittle (anti-pattern #1)
- **C.** Structured results use fewer tokens than any prose
- **D.** Prose cannot be returned by subagents

**Answer:** `B`

**Explanation:** Verifying that each subagent delivered a valid, contract-conforming structured output is a deterministic, checkable completion criterion. Inferring completion from free text is anti-pattern #1 and fails when wording varies. The advantage is reliability/determinism, not a token claim, and subagents certainly can return prose.

**Tags:** `Domain 1` `Advanced` `S3` `Termination`

### Q64. An orchestrator in S3 must decide how many subagents to spawn for a query. Which heuristic aligns with Anthropic's guidance?

- **A.** Always spawn the maximum the system allows
- **B.** Scale the number of subagents to the query's actual breadth/complexity, since each subagent adds token cost and coordination overhead
- **C.** Always spawn exactly one regardless of breadth
- **D.** Spawn one subagent per word in the query

**Answer:** `B`

**Explanation:** Anthropic's research-system guidance is to match subagent count to the query's real breadth — over-spawning multiplies cost and coordination overhead with diminishing returns, while under-spawning misses breadth. Fixed maxima, a fixed single agent, or absurd proxies like word count ignore the actual task structure.

**Tags:** `Domain 1` `Advanced` `S3` `Orchestrator-Workers`

### Q65. A workflow chains step A → step B, but B sometimes receives malformed output from A. What is the most robust fix within the prompt-chaining pattern?

- **A.** Remove the gate between steps and hope B handles it
- **B.** Add a programmatic validation gate between A and B that checks/repairs A's output (or re-runs A) before B consumes it
- **C.** Merge A and B into one giant prompt with no checks
- **D.** Escalate to a human on every run

**Answer:** `B`

**Explanation:** Prompt chaining explicitly supports programmatic "gates" between steps to validate intermediate outputs before the next step proceeds — that is the pattern's reliability mechanism. Removing the gate propagates corruption; merging into one prompt loses the checkpoint and verifiability; escalating every run defeats automation.

**Tags:** `Domain 1` `Intermediate` `Workflow Patterns` `Reliability`

### Q66. Which design correctly separates "author" from "reviewer" to avoid same-session self-review bias in S5 multi-pass review?

- **A.** Have the same agent in the same context critique its own output
- **B.** Run the review in a fresh context or a separate agent that does not inherit the author's reasoning chain
- **C.** Ask the author to lower its temperature and re-read
- **D.** Skip review entirely to save cost

**Answer:** `B`

**Explanation:** A reviewer that shares the author's context inherits its assumptions and blind spots — anti-pattern #9. A fresh context or a distinct agent reviews with independent reasoning, catching issues the author rationalized away. Re-reading at lower temperature does not remove the shared bias, and skipping review abandons quality control.

**Tags:** `Domain 1` `Intermediate` `S5` `Self-Review Anti-Pattern`

### Q67. For an open-ended customer issue in S1 that may need an unpredictable sequence of lookups, refunds, and escalations, which architecture is most appropriate?

- **A.** A rigid linear prompt chain with no branching
- **B.** An agent (dynamic tool use driven by stop_reason) with a curated toolset and deterministic policy guardrails, falling back to human escalation on authority boundaries
- **C.** A pure parallelization fan-out
- **D.** A single one-shot completion with no tools

**Answer:** `B`

**Explanation:** Unpredictable, branching control flow is the signature of a true agent: it should drive tool use via stop_reason, use a tight curated toolset, enforce policy in code, and escalate on authority limits. A rigid chain cannot adapt; parallelization is for independent slices, not adaptive sequencing; a tool-less one-shot cannot act.

**Tags:** `Domain 1` `Advanced` `S1` `Architecture Selection`

### Q68. Which of the following correctly orders the "increasing complexity" ladder Anthropic suggests you climb only as needed?

- **A.** Multi-agent → agent → workflow → single prompt
- **B.** Single LLM call → workflow → single agent → multi-agent system
- **C.** Multi-agent → single prompt → workflow → agent
- **D.** They are unordered; pick any

**Answer:** `B`

**Explanation:** The complexity ladder runs from the simplest (a single well-crafted LLM call), to a structured workflow, to a single autonomous agent, to a multi-agent system — climbing only when the task demands it. The reversed orders invert the simplicity-first principle, and the ladder is not arbitrary. This ordering is a frequent exam signal.

**Tags:** `Domain 1` `Foundational` `Architecture Selection`

### Q69. A subagent in S3 keeps re-deriving facts the coordinator already established, wasting tokens. What design change helps most?

- **A.** Give the subagent the entire coordinator context to avoid re-derivation
- **B.** Pass the subagent a concise, relevant briefing (the established facts and its precise objective) so it starts from known context without inheriting irrelevant clutter
- **C.** Remove all context so the subagent works from scratch
- **D.** Increase the subagent's iteration cap

**Answer:** `B`

**Explanation:** Effective delegation passes a focused briefing — the relevant known facts plus a precise objective — so the subagent avoids redundant work while keeping its context lean. Dumping the whole coordinator context reintroduces clutter and bloat that isolation was meant to prevent; zero context guarantees re-derivation; a bigger cap just lets it waste more.

**Tags:** `Domain 1` `Advanced` `S3` `Subagent Isolation`

### Q70. What is the single best summary of how a well-architected agent decides to keep going versus stop?

- **A.** Keep going until the text says "done," then stop
- **B.** Continue while stop_reason is tool_use (executing tools and feeding results back) and task goals remain unmet; stop on end_turn or a satisfied goal/policy check, bounded by a safety iteration backstop
- **C.** Stop at a fixed number of iterations, always
- **D.** Stop when the customer's sentiment turns positive

**Answer:** `B`

**Explanation:** Correct termination is driven by the deterministic stop_reason plus an explicit goal/policy check, with an iteration cap only as a safety backstop. Parsing "done" is anti-pattern #1; a fixed iteration count as the primary control is anti-pattern #2; sentiment is anti-pattern #5 and unrelated to task completion.

**Tags:** `Domain 1` `Intermediate` `Termination`

### Q71. In S1, the agent must call a refund tool that is occasionally rate-limited (HTTP 429). What is the best loop-level handling?

- **A.** Treat 429 as a permanent failure and escalate to a human immediately
- **B.** Respect the retry timing (honor Retry-After / use exponential backoff with jitter) and retry within a bounded budget before escalating
- **C.** Retry instantly in a tight loop until it succeeds
- **D.** Return a fake success so the conversation continues

**Answer:** `B`

**Explanation:** A 429 is a transient, retryable signal: honor the server's retry guidance or back off exponentially with jitter and retry within a bounded budget, escalating only if it persists. Immediate escalation is wasteful for a transient hiccup; tight instant retries worsen the rate limit; fake success is anti-pattern #7 and could imply an unprocessed refund.

**Tags:** `Domain 1` `Intermediate` `S1` `Fault Tolerance`

### Q72. Which is the strongest reason to prefer a workflow over an agent for an S5 CI/CD review gate that must behave identically across thousands of PRs?

- **A.** Workflows can use more tools than agents
- **B.** Predefined workflows give deterministic, repeatable behavior and easier auditing — exactly what a CI gate needs — without the variance of open-ended agency
- **C.** Agents are not allowed in CI/CD
- **D.** Workflows are the only thing that can call the Batch API

**Answer:** `B`

**Explanation:** A CI/CD gate values consistency, reproducibility, and auditability, which a predefined workflow delivers; open-ended agency introduces unnecessary variance across runs. Tool count is not the differentiator, agents are not banned in CI, and both workflows and agents can call the Batch API. Match the architecture to the consistency requirement.

**Tags:** `Domain 1` `Advanced` `S5` `Architecture Selection`

### Q73. An orchestrator-worker design returns per-document extraction results, but the team reports only an overall accuracy of 94%. What reliability concern does this raise for D1/D5?

- **A.** None; 94% aggregate is sufficient on its own
- **B.** Aggregate accuracy can mask poor performance on specific document types or categories (anti-pattern #10); evaluate per slice to catch hidden failures
- **C.** The orchestrator should be replaced by a single prompt
- **D.** The team should raise the iteration cap to 100

**Answer:** `B`

**Explanation:** A single aggregate number can hide that one document type performs at, say, 40% while others inflate the mean — anti-pattern #10. Per-slice (per-document-type/category) evaluation surfaces these failures so the architecture can be fixed where it actually breaks. Switching to one prompt or bumping the cap does not address the masked-failure problem.

**Tags:** `Domain 1` `Advanced` `S6` `Reliability Evaluation`

### Q74. Which orchestration choice best minimizes latency for an S3 query whose subtopics are fully independent and equally hard?

- **A.** Sequentially process each subtopic one after another
- **B.** Fan out the independent subtopics to parallel subagents and synthesize their results, since independence enables concurrency
- **C.** Use evaluator-optimizer looping on a single agent
- **D.** Use prompt chaining across all subtopics in series

**Answer:** `B`

**Explanation:** When subtopics are independent, fanning them out to parallel subagents collapses total latency to roughly the slowest branch instead of the sum of all branches. Sequential processing and series chaining needlessly serialize independent work; evaluator-optimizer is a quality-feedback loop, not a parallel-breadth mechanism.

**Tags:** `Domain 1` `Advanced` `S3` `Cost/Latency Trade-offs`

### Q75. A junior engineer proposes controlling an agent purely by "asking it in the system prompt to call tools in the right order and stop when finished." What is the most accurate critique?

- **A.** It is a solid, production-grade control mechanism
- **B.** It conflates control with prompting: ordering/stop guarantees should come from the loop's stop_reason handling and code-level orchestration, not from prompt instructions alone (anti-patterns #1 and #3)
- **C.** It only fails because the prompt is too short
- **D.** It is fine as long as temperature is 0

**Answer:** `B`

**Explanation:** Relying on the prompt to guarantee tool ordering and termination mixes two anti-patterns: trusting natural-language self-stopping (#1) and prompt-based enforcement of critical control flow (#3). Real control comes from deterministic loop logic that branches on stop_reason and orchestrates tool calls in code. Prompt length and temperature 0 do not convert prompting into a guarantee.

**Tags:** `Domain 1` `Advanced` `Agent Loop` `Deterministic Enforcement`
