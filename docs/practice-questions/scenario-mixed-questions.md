# CCA-F Scenario-Anchored Mixed Question Bank

These are original practice questions written for personal self-study of the **Claude Certified Architect – Foundations (CCA-F)** exam. They are NOT real or proctored exam items. Every question is anchored to one of the six published production scenarios (S1–S6), mirroring the real exam format where each item presents a short situation and asks for the best architectural decision. Distractors deliberately use the ten high-yield anti-patterns so you learn to recognize them. Source behavior references: platform.claude.com/docs, docs.anthropic.com/en/docs/claude-code, modelcontextprotocol.io, and Anthropic engineering writing on building effective agents.

How to study: read the situation, pick before peeking, then read the Explanation. The Explanation always says why the key is right AND why the most tempting wrong option fails, naming the anti-pattern where relevant. Tags identify the domain (D1–D5), scenario (S1–S6), and difficulty.

## Scenario 1 — Customer Support Resolution Agent

S1 context: An Agent SDK loop resolves customer tickets. It calls MCP tools (order lookup, refund, knowledge base search), decides when an issue is resolved, and escalates to a human when it cannot safely handle the case. Reliability, correct loop termination, and trustworthy escalation are the central concerns.

### Q1. The support agent must stop its tool-use loop when a ticket is fully resolved. Which signal should the orchestrator use to decide the turn is complete?
A) Scan the model's text for phrases like "I'm done" or "issue resolved" and break the loop
B) Check the API stop_reason; continue while it is tool_use and finish when it is end_turn
C) Break after a fixed maximum of 5 iterations regardless of state
D) Ask the model to emit a self-reported confidence score and stop above 0.9
Answer: B
Explanation: The Messages API returns a stop_reason; an agentic loop continues while stop_reason is tool_use (execute tools, append results, call again) and ends naturally on end_turn. Parsing natural language like "I'm done" (A) is the classic termination anti-pattern because phrasing is non-deterministic. Iteration caps (C) are a safety backstop, not the primary control, and confidence scores (D) are not reliable stop signals.
Tags: D1, S1, Foundational

### Q2. Refunds above $500 require a documented policy check that must never be skipped. How should the architecture enforce this rule?
A) Add a strongly worded instruction in the system prompt telling the model to always verify policy before large refunds
B) Implement the threshold check in deterministic code (a tool wrapper or hook) that the refund cannot bypass
C) Trust the model's reasoning since it has seen the policy in context
D) Lower the temperature so the model follows instructions more strictly
Answer: B
Explanation: Critical business rules must be enforced deterministically in code (a hook or a guarded tool implementation) so they cannot be skipped, regardless of how the model reasons. Prompt-based enforcement (A) is the listed anti-pattern: prompts express intent but provide no guarantee. Lowering temperature (D) reduces variability but never guarantees a rule is applied.
Tags: D2, S1, Intermediate

### Q3. The team wants the agent to escalate complex tickets to a human. Which escalation trigger is architecturally sound?
A) Escalate when the customer's message sentiment is negative or angry
B) Escalate when the model reports low confidence in its own answer
C) Escalate on objective conditions: a required tool failed, the action exceeds an authority threshold, or required data is missing
D) Escalate whenever the conversation exceeds three turns
Answer: C
Explanation: Escalation should key on objective, observable conditions tied to task feasibility and authority limits. Sentiment-based escalation (A) conflates emotion with complexity — an angry customer may have a trivial request. Self-reported confidence (B) is an unreliable proxy. A flat turn count (D) is arbitrary and unrelated to whether the agent can safely resolve the case.
Tags: D1, S1, Intermediate

### Q4. An order-lookup MCP tool times out against the upstream system. What should the tool return to the model?
A) An empty result set so the agent can keep going smoothly
B) A generic "An error occurred" string with no detail
C) A structured error describing what failed (timeout), which system, and whether retry is sensible
D) Silently swallow the failure and report success
Answer: C
Explanation: Tool errors should carry diagnostic context so the model can reason about retrying, switching tools, or escalating. Returning empty results (A) or silently reporting success (D) is the "silent suppression" anti-pattern that causes the agent to act on false premises. Generic messages (B) hide the diagnostic context the model needs to recover.
Tags: D2, S1, Intermediate

### Q5. The agent currently exposes 18 tools (lookup, refund, KB search, CRM update, billing, shipping, etc.) and tool-selection accuracy is dropping. What is the best remedy?
A) Keep all 18 tools but add longer descriptions so the model picks better
B) Curate a tight core set (~4–7) and move specialized capabilities behind subagents or separate MCP boundaries
C) Raise max_tokens so the model has room to consider every tool
D) Add a prompt instruction listing which tool to use for each case
Answer: B
Explanation: Too many tools per agent degrades selection accuracy; the fix is a curated set of roughly four to seven tools, with specialized capabilities split into subagents or separate MCP servers. Longer descriptions (A) and prompt lists (D) do not address the core overload problem and add more to reason over. Token budget (C) is unrelated to selection accuracy.
Tags: D2, S1, Advanced

### Q6. During a refund flow the model produced a tool_use block. What must the next API request contain to keep the loop valid?
A) Only the assistant's text, dropping the tool_use block to save tokens
B) The assistant turn including the tool_use block, followed by a user turn containing a matching tool_result with the same tool_use_id
C) A fresh conversation with no prior context
D) A system prompt restating the tool schema
Answer: B
Explanation: The protocol requires you to append the assistant message containing the tool_use, then a user message with a tool_result carrying the matching tool_use_id, before the next call. Dropping the tool_use block (A) breaks the pairing and the API rejects it. Starting fresh (C) loses the in-progress task; restating the schema (D) is unnecessary and does not satisfy the pairing requirement.
Tags: D2, S1, Foundational

### Q7. Support wants faster responses for repeated KB context (a large policy document sent on every ticket). What feature reduces cost and latency here?
A) Prompt caching with a cache breakpoint on the stable policy document prefix
B) Lowering max_tokens on every request
C) Switching to the Batch API for real-time chat
D) Truncating the policy document to fit fewer tokens
Answer: A
Explanation: Prompt caching lets you mark a stable prefix (the large policy document) so subsequent requests reuse it at reduced cost and latency. The Batch API (C) is for asynchronous, non-interactive workloads, not live chat. Lowering max_tokens (B) and truncating the document (D) hurt answer quality without addressing repeated-prefix reuse.
Tags: D5, S1, Intermediate

### Q8. The agent should never refund an order it could not first verify exists. How do you guarantee ordering of tool calls?
A) Tell the model in the prompt to "always look up the order before refunding"
B) Make the refund tool's implementation verify the order server-side and reject if unverified
C) Set a high iteration cap so there is time to look up the order
D) Use sentiment analysis to gate the refund
Answer: B
Explanation: Ordering guarantees for a critical action belong in deterministic code: the refund tool should verify the order server-side and refuse otherwise. Prompt sequencing (A) is the prompt-enforcement anti-pattern and gives no guarantee. Iteration caps (C) and sentiment (D) are unrelated to enforcing a precondition.
Tags: D2, S1, Advanced

### Q9. You need to evaluate the support agent's quality before launch. Which evaluation approach best surfaces real risk?
A) A single aggregate resolution-accuracy number across all ticket types
B) Per-category evaluation (refunds, shipping, billing, account) with separate pass thresholds
C) Counting how many tickets the agent marked "resolved"
D) Asking the agent to grade its own transcripts
Answer: B
Explanation: Aggregate accuracy can mask a category that fails badly (e.g., refunds at 60% hidden inside a 92% overall). Per-slice evaluation exposes that failure — this is the anti-pattern about aggregate metrics masking per-category failures. Self-marked resolution (C) and self-grading (D) are unreliable because they inherit the agent's own bias.
Tags: D4, S1, Advanced

### Q10. The escalation handoff must give the human agent everything they need. What is the best handoff payload design?
A) Just the original customer message
B) A structured summary: the issue, actions already taken, tool results, why escalation triggered, and remaining options
C) The full raw token-by-token transcript with no structure
D) Only the model's final confidence score
Answer: B
Explanation: A useful handoff is a structured summary capturing the issue, steps taken, tool outcomes, the objective trigger, and open options, so the human resumes quickly. The bare original message (A) and a confidence score (D) omit the work done. A raw unstructured transcript (C) forces the human to re-derive everything, defeating the purpose of escalation.
Tags: D1, S1, Intermediate

## Scenario 2 — Code Generation with Claude Code

S2 context: A team uses Claude Code to implement features in a real repository. They rely on CLAUDE.md project memory, plan mode for risky changes, custom slash commands, and permission settings. The focus is configuring Claude Code so it behaves predictably and safely inside the codebase.

### Q11. Where should durable, repo-specific conventions (build commands, code style, do-not-touch files) live so every Claude Code session picks them up?
A) Pasted into the chat at the start of each session
B) In a CLAUDE.md file committed at the project root
C) Only in the developer's shell history
D) In a code comment in a random source file
Answer: B
Explanation: CLAUDE.md is project memory that Claude Code loads automatically, making it the right home for durable conventions shared across sessions and teammates. Pasting per session (A) is not durable and is easy to forget. Shell history (C) and a stray comment (D) are not discovered by Claude Code as project context.
Tags: D3, S2, Foundational

### Q12. A developer is about to ask Claude Code to refactor authentication across many files. What is the safest first step?
A) Let it edit immediately and review the diff afterward
B) Use plan mode so Claude proposes an approach for review before making any edits
C) Increase the iteration cap so it can finish in one pass
D) Disable permission prompts to avoid interruptions
Answer: B
Explanation: Plan mode has Claude Code research and propose a plan without editing, which is the safe entry point for a wide, risky change so a human can approve the approach first. Editing immediately (A) risks sweeping unwanted changes. Raising iteration caps (C) and disabling permission prompts (D) remove safeguards exactly when you want them most.
Tags: D3, S2, Intermediate

### Q13. The team repeats the same multi-step "open a PR with tests and changelog" workflow. How should they encode it for reuse?
A) Write it once in CLAUDE.md as a paragraph of prose
B) Create a custom slash command (a reusable prompt template) in the project's commands directory
C) Ask each developer to memorize the steps
D) Hardcode it into a single Python script unrelated to Claude Code
Answer: B
Explanation: Custom slash commands are reusable prompt templates stored in the project (e.g., under .claude/commands) and invoked by name, ideal for a repeatable workflow. A CLAUDE.md paragraph (A) informs but is not an invokable command. Memorization (C) is not reusable tooling, and an unrelated script (D) does not integrate with the Claude Code session.
Tags: D3, S2, Intermediate

### Q14. CLAUDE.md has grown to thousands of lines covering many unrelated subsystems. What problem does this create and what is the fix?
A) No problem; more context is always better
B) It bloats every session's context and dilutes attention; trim to high-signal rules and use imports/subdirectory CLAUDE.md files for local detail
C) It will crash Claude Code, so delete CLAUDE.md entirely
D) Move everything into the system prompt instead
Answer: B
Explanation: An oversized CLAUDE.md consumes context budget on every turn and dilutes the model's attention; keep it concise and high-signal, pushing local detail into nested or imported memory. "More is always better" (A) ignores context-budget limits. Deleting it (C) loses valuable guidance, and there is no separate user-controlled system prompt to dump it into (D).
Tags: D5, S2, Advanced

### Q15. A custom command must run the project's test suite the exact same way every time, with no chance the model "decides" to skip it. Where does that guarantee come from?
A) A polite instruction in the slash command prompt
B) A hook (e.g., a PostToolUse or Stop hook) that runs tests deterministically as part of the workflow
C) Telling the model the tests are very important
D) A higher temperature so the model is more thorough
Answer: B
Explanation: Deterministic guarantees come from Claude Code hooks that execute shell commands at defined lifecycle points, independent of model choice. Instructions in a prompt (A and C) are the prompt-enforcement anti-pattern and can be skipped. Temperature (D) changes sampling, not whether a step deterministically runs.
Tags: D3, S2, Advanced

### Q16. When should a developer add a tool/command to the permission allowlist versus leaving prompts on?
A) Allowlist destructive commands like force-push and rm -rf to move faster
B) Allowlist frequent, low-risk, read-only operations; keep prompts for destructive or irreversible actions
C) Disable all prompts globally for every project
D) Never allowlist anything; always confirm everything
Answer: B
Explanation: Good permission hygiene allowlists frequent low-risk read-only actions to cut friction while keeping confirmation for destructive or irreversible operations. Allowlisting destructive commands (A) and disabling all prompts (C) remove safety where it matters most. Confirming literally everything (D) creates so much friction that users start rubber-stamping prompts.
Tags: D3, S2, Intermediate

### Q17. Claude Code keeps editing a generated file that must never be hand-modified. What is the cleanest fix?
A) Hope the model remembers not to touch it
B) Document the do-not-edit rule in CLAUDE.md and back it with a hook or permission rule that blocks edits to that path
C) Delete the file so it cannot be edited
D) Lower max_tokens so it writes less
Answer: B
Explanation: Combine a clear CLAUDE.md note for intent with a deterministic guard (a PreToolUse hook or permission deny rule) so edits to the protected path are actually prevented. Relying on memory (A) is the prompt-enforcement trap. Deleting the file (C) breaks the build, and token limits (D) are unrelated to which paths get edited.
Tags: D3, S2, Advanced

### Q18. A long Claude Code session is drifting and the context is cluttered with stale exploration. What is the right move?
A) Keep going and hope it self-corrects
B) Clear or compact the context and restart the task with a focused, well-scoped prompt
C) Raise the iteration cap so it has more attempts
D) Switch to a smaller model to save tokens
Answer: B
Explanation: When context is cluttered and the session drifts, clearing/compacting and restarting with a tight prompt restores focus — managing the context window is a first-class reliability lever. Continuing (A) compounds drift. More iterations (C) on a polluted context just produces more noise, and a smaller model (D) does not fix context hygiene.
Tags: D5, S2, Intermediate

### Q19. The team wants Claude Code to follow a specific commit-message format on every commit. Best mechanism?
A) Add the exact format to CLAUDE.md so the model applies it consistently
B) Only fix the message manually after each commit
C) Put the format in a one-off chat message each time
D) Assume the model already knows your preferred format
Answer: A
Explanation: Stable, repo-wide conventions like a commit-message format belong in CLAUDE.md so every session applies them. Manual fixes (B) defeat automation. A per-chat message (C) is not durable, and assuming the model infers your house style (D) is unreliable. (For a hard guarantee you could additionally add a commit hook, but among these options CLAUDE.md is the correct configuration home.)
Tags: D3, S2, Foundational

### Q20. Claude Code proposes a plan in plan mode that looks mostly right but touches a file you flagged as sensitive. Best response?
A) Approve as-is to keep momentum
B) Reject/adjust the plan, restate the constraint, and have it replan before any edits
C) Let it edit and revert later if needed
D) Increase permissions so it can finish without prompts
Answer: B
Explanation: Plan mode exists precisely so you can correct course before edits; reject or refine the plan and restate the constraint so it replans. Approving a flawed plan (A) or editing-then-reverting (C) wastes work and risks the sensitive file. Raising permissions (D) removes a guardrail when the plan is already suspect.
Tags: D3, S2, Intermediate

## Scenario 3 — Multi-Agent Research System

S3 context: A coordinator agent decomposes a research question into subtasks, dispatches subagents to investigate in parallel, and synthesizes their findings. Reliability hinges on clear delegation, isolated context per subagent, robust error handling, and correct aggregation.

### Q21. In an orchestrator-worker research system, what is the coordinator's primary responsibility?
A) Doing all the web searches itself in one giant context
B) Decomposing the question into well-scoped subtasks, dispatching subagents, and synthesizing their results
C) Reviewing its own answer in the same context to save agents
D) Maximizing the number of subagents regardless of the task
Answer: B
Explanation: In the orchestrator-worker pattern the coordinator plans, delegates scoped subtasks, and synthesizes — each subagent works in its own context window. Doing everything itself (A) defeats the point and overloads one context. Same-context self-review (C) inherits the author's bias, and spawning subagents without need (D) adds coordination cost and tokens.
Tags: D1, S3, Intermediate

### Q22. Why give each research subagent its own separate context window rather than sharing one?
A) It is required by the billing model
B) Isolation prevents one subtask's exploration from polluting another's reasoning and lets each focus its full budget on its slice
C) It guarantees the subagents will agree with each other
D) It removes the need for any synthesis step
Answer: B
Explanation: Separate contexts give each subagent a clean, focused window so unrelated exploration does not crowd out or bias another subtask, and each gets full budget for its slice. It is an architectural choice, not a billing requirement (A). Isolation does not force agreement (C), and you still must synthesize the independent findings (D).
Tags: D5, S3, Intermediate

### Q23. One subagent's search tool fails repeatedly. How should the system handle it?
A) Return empty findings and let the coordinator treat the topic as fully covered
B) Surface a structured error to the coordinator so it can retry, reassign, or note the gap in the synthesis
C) Have the subagent claim success to keep the pipeline flowing
D) Abort the entire research run on any single subagent failure
Answer: B
Explanation: A failed subtask should propagate a structured error so the coordinator can retry, reassign, or explicitly flag the gap — keeping the final report honest. Empty-as-success (A) and claiming success (C) are the silent-suppression anti-pattern that yields confidently incomplete answers. Aborting everything (D) is too brittle when other subtasks succeeded.
Tags: D1, S3, Advanced

### Q24. How should the coordinator decide it has gathered enough to synthesize and stop dispatching subagents?
A) When a subagent's text says "we have enough information now"
B) When the planned subtasks report back (success or explicit gap) and coverage of the question's components is met
C) After a hardcoded 10 subagents, always
D) When the coordinator's self-reported confidence crosses a threshold
Answer: B
Explanation: Stopping should be driven by completion of the planned subtasks and coverage of the question's components, with gaps explicitly recorded — a deterministic, plan-based criterion. Parsing "we have enough" (A) is the natural-language termination anti-pattern. A fixed subagent count (C) is an arbitrary cap, and self-reported confidence (D) is an unreliable control signal.
Tags: D1, S3, Advanced

### Q25. When is a multi-agent architecture justified over a single agent for a task?
A) Always — more agents are strictly better
B) When the task has parallelizable, separable subtasks whose isolation outweighs coordination and token overhead
C) Only when the single agent has exactly one tool
D) Never, because coordination is too hard
Answer: B
Explanation: Multi-agent designs pay off when subtasks are genuinely separable and parallelizable enough that isolation benefits exceed the added coordination and token cost. "Always more agents" (A) ignores real overhead; tool count alone (C) is not the trigger; and "never" (D) discards a pattern that wins for breadth-heavy research.
Tags: D1, S3, Intermediate

### Q26. A subagent returns a long raw dump. What should it hand back to the coordinator to keep aggregation clean?
A) The entire unfiltered transcript and every tool output
B) A concise, structured findings summary with sources, so the coordinator's context stays focused
C) Only a one-word "done"
D) Its self-assigned confidence score and nothing else
Answer: B
Explanation: Subagents should compress their work into a structured findings summary (with sources) so the coordinator aggregates signal rather than drowning in raw tokens. Dumping the full transcript (A) blows the coordinator's context budget — the very thing isolation was meant to protect. "done" (C) and a lone confidence score (D) carry no usable findings.
Tags: D5, S3, Advanced

### Q27. The coordinator must verify research claims before publishing. Which review setup avoids confirmation bias?
A) The same agent that wrote the synthesis reviews its own claims
B) A separate verification agent with fresh context checks claims against sources
C) Trust the synthesis because it cited sources
D) Ask the synthesis agent to rate its own confidence and publish if high
Answer: B
Explanation: A fresh-context verification agent avoids inheriting the author's reasoning, catching unsupported claims the original would rationalize — the antidote to the same-session self-review anti-pattern (A and effectively D). Merely citing sources (C) does not confirm the claim actually follows from them; independent verification does.
Tags: D1, S3, Advanced

### Q28. Two subagents return contradictory findings. What is the right coordinator behavior?
A) Pick one at random to keep things simple
B) Surface the conflict, weigh source quality/recency, and either reconcile with evidence or report the disagreement transparently
C) Average the two answers into a single number
D) Suppress both and report nothing on that point
Answer: B
Explanation: Contradictions should be made explicit and adjudicated on source quality and recency, or reported transparently if unresolved. Random selection (A) and averaging unlike claims (C) destroy information. Suppressing both (D) is the silent-suppression anti-pattern and leaves a hidden gap in the report.
Tags: D1, S3, Advanced

### Q29. To keep total cost bounded in the research system, which control is appropriate as a safety backstop (not the primary logic)?
A) A maximum subagent / iteration ceiling that triggers a graceful wrap-up if exceeded
B) Parsing model text to decide when money runs out
C) Setting temperature to 0 to reduce cost
D) Removing all error handling to shorten runs
Answer: A
Explanation: A maximum subagent or iteration ceiling is a legitimate safety backstop that forces a graceful wrap-up if normal completion logic does not fire — but it is a backstop, not the primary stop control. Parsing text for budget (B) is unreliable; temperature (C) does not bound cost; and removing error handling (D) trades safety for speed and corrupts results.
Tags: D1, S3, Intermediate

### Q30. The coordinator's prompt should describe subtasks to subagents. What makes delegation effective?
A) Vague one-liners so subagents have freedom
B) Clear scope, the specific output format expected, and the boundary of what NOT to investigate
C) The full coordinator context copied into every subagent
D) A demand that each subagent return maximum tokens
Answer: B
Explanation: Effective delegation gives each subagent a clear objective, the expected output shape, and explicit boundaries, so results compose cleanly. Vague tasks (A) cause overlap and drift. Copying the coordinator's full context (C) wastes the isolation benefit, and demanding maximum tokens (D) bloats aggregation without adding signal.
Tags: D1, S3, Intermediate

## Scenario 4 — Developer Productivity with Claude

S4 context: An individual developer uses Claude (Claude Code plus MCP servers) to explore an unfamiliar codebase, answer questions about it, and make targeted changes. The emphasis is on using built-in tools, connecting the right MCP servers, and efficient codebase navigation.

### Q31. A developer needs to understand an unfamiliar large repository fast. What is the most effective first approach with Claude Code?
A) Paste the entire repository into one prompt
B) Let Claude Code use its built-in search/read tools to explore on demand, starting from entry points and READMEs
C) Ask it to guess the architecture without reading files
D) Open every file manually and summarize each yourself
Answer: B
Explanation: Claude Code's built-in file and search tools let it navigate on demand, pulling only relevant files into context — the efficient way to map a large repo. Pasting everything (A) overflows and dilutes context. Guessing (C) yields hallucinated architecture, and manual summarizing of every file (D) discards the assistant's value.
Tags: D4, S4, Foundational

### Q32. The developer wants Claude to query the team's issue tracker and CI system during a session. What is the right integration mechanism?
A) Add MCP servers for those systems and expose only the needed tools
B) Paste screenshots of the issue tracker into chat
C) Ask the model to recall the issue tracker from training data
D) Give it 18 unrelated MCP servers at once so everything is available
Answer: A
Explanation: MCP servers are the standard way to connect external systems (issue tracker, CI) and expose a focused set of tools to the agent. Screenshots (B) are lossy and manual; training data (C) cannot know your live issues; and wiring up many unrelated servers (D) recreates the too-many-tools overload that hurts selection accuracy.
Tags: D2, S4, Intermediate

### Q33. Which describes the Model Context Protocol (MCP) most accurately?
A) A proprietary Anthropic-only API for a single product
B) An open protocol that standardizes how applications expose tools, resources, and prompts to LLM clients
C) A replacement for the Messages API
D) A library that only works inside Claude Code
Answer: B
Explanation: MCP is an open standard (modelcontextprotocol.io) for connecting models to external tools, resources, and prompts via a common client-server protocol, usable across many clients. It is not single-product or Anthropic-locked (A, D) and does not replace the Messages API (C) — it complements model interaction by standardizing context/tool access.
Tags: D2, S4, Foundational

### Q34. The developer asks Claude to "find where rate limiting is implemented." What tool behavior should you expect and want?
A) It answers from memory without opening files
B) It searches the codebase (grep/glob), opens the matching files, and cites the concrete locations it found
C) It rewrites the rate limiter immediately
D) It asks you to paste the whole codebase first
Answer: B
Explanation: The desired behavior is grounded exploration: search, read the matches, and cite real file locations, so the answer is verifiable. Answering from memory (A) risks hallucinating files that do not exist. Rewriting on a "find" request (C) overreaches the ask, and demanding the whole codebase (D) is unnecessary when search tools exist.
Tags: D4, S4, Foundational

### Q35. An MCP tool the developer relies on returns ambiguous, untyped output that the model often misreads. Best fix at the tool boundary?
A) Tell the model in the prompt to be careful reading it
B) Improve the MCP tool to return a clear, structured, typed result with field names and units
C) Increase max_tokens so the model reads more carefully
D) Add more tools so the model has alternatives
Answer: B
Explanation: Ambiguity should be fixed at the tool boundary by returning structured, well-named, typed output the model can parse reliably — good tool design is a first-class architectural concern. Prompt pleading (A) is the enforcement anti-pattern. Token budget (C) does not clarify ambiguous data, and adding tools (D) increases overload without fixing the source.
Tags: D2, S4, Intermediate

### Q36. The developer worries an MCP server could perform destructive writes. What is the prudent configuration?
A) Grant all tools full write access for convenience
B) Expose read-only tools by default and gate write/destructive tools behind explicit permission
C) Disable MCP entirely and copy-paste data manually
D) Trust the prompt to tell the model not to delete things
Answer: B
Explanation: Least-privilege at the tool layer means defaulting to read-only and requiring explicit approval for write or destructive operations. Full write access (A) maximizes blast radius. Abandoning MCP (C) throws away the integration's value, and a prompt instruction (D) is the unenforceable prompt anti-pattern for a safety-critical control.
Tags: D2, S4, Intermediate

### Q37. To answer questions about a 500k-line monorepo efficiently, how should context be managed across a session?
A) Load the whole repo once and keep it resident
B) Pull only relevant files into context as needed and drop them when done, keeping the window focused
C) Never read files; reason abstractly
D) Restart the session after every single question
Answer: B
Explanation: Efficient large-repo work means retrieving just-relevant files on demand and keeping the window lean — context is a scarce budget. Keeping the whole repo resident (A) is impossible and would crowd out reasoning. Abstract reasoning (C) hallucinates, and restarting after every question (D) throws away useful continuity.
Tags: D5, S4, Intermediate

### Q38. The developer wants reproducible answers about the codebase that a teammate can verify. What should Claude include?
A) Confident prose with no references
B) Concrete file paths, symbol names, and line references for each claim
C) A self-reported confidence score per answer
D) Only a high-level summary with no specifics
Answer: B
Explanation: Verifiable answers cite concrete paths, symbols, and lines so a teammate can check them directly. Unreferenced prose (A) and summary-only (D) are not reproducible. A confidence score (C) is a self-report that conveys feeling, not evidence, and is the confidence-as-signal anti-pattern.
Tags: D4, S4, Foundational

### Q39. Which is the better division of labor when a task needs both repo exploration and a specialized data-analysis capability?
A) Stuff every tool into one agent so nothing is missed
B) Keep the main agent's toolset tight and delegate the specialized analysis to a subagent or dedicated MCP server
C) Remove the analysis capability to simplify
D) Ask the model to simulate the analysis tool in its head
Answer: B
Explanation: Keeping the primary agent focused and pushing specialized work to a subagent or dedicated MCP server preserves tool-selection accuracy and clean boundaries. Cramming all tools into one agent (A) is the too-many-tools anti-pattern. Removing the capability (C) fails the task, and simulating a tool mentally (D) produces ungrounded results.
Tags: D2, S4, Advanced

### Q40. Claude reports it could not find a referenced config file. What is the correct behavior versus an anti-pattern?
A) Invent plausible config contents so the answer looks complete
B) Report the file was not found, where it looked, and ask whether the path is correct or generated elsewhere
C) Silently return an empty config and proceed
D) Claim success and move on
Answer: B
Explanation: The correct behavior is transparent failure reporting with the search locations and a clarifying question. Inventing contents (A) is hallucination; returning empty (C) and claiming success (D) are the silent-suppression anti-pattern that leads the developer to act on false information.
Tags: D4, S4, Intermediate

## Scenario 5 — Claude Code for CI/CD

S5 context: Claude runs inside CI to review pull requests at scale. It produces structured findings the pipeline can act on (pass/fail, severity), uses the Batch API for large nightly review jobs, and performs multi-pass review with fresh context to avoid bias. Determinism and machine-parseable output are key.

### Q41. The CI pipeline must programmatically gate merges on Claude's review output. How should that output be produced?
A) Free-form prose that a regex tries to interpret
B) A structured/JSON output (via tool_use or structured outputs) with explicit fields like severity and pass/fail
C) A single emoji indicating sentiment
D) The model saying "looks good to me" in plain text
Answer: B
Explanation: Pipeline gating needs machine-parseable, schema-conformant output (tool_use or structured outputs) with explicit decision fields the CI can branch on. Prose-plus-regex (A) is brittle and breaks on phrasing changes. An emoji (C) or "looks good" (D) is unstructured and not safely parseable for an automated gate.
Tags: D4, S5, Intermediate

### Q42. The team runs a large nightly review across thousands of diffs with no need for instant responses. Which API fits best?
A) Synchronous Messages API, one slow request at a time
B) The Batch API for high-volume asynchronous jobs at lower cost
C) Streaming responses to a human reviewer all night
D) Prompt caching alone with no batching
Answer: B
Explanation: The Batch API is built for high-volume, latency-tolerant jobs processed asynchronously at reduced cost — exactly the nightly bulk-review case. One-at-a-time synchronous calls (A) are slow and costly at that scale. Streaming to a human (C) defeats automation, and caching (D) helps with repeated prefixes but is not a batching mechanism.
Tags: D5, S5, Intermediate

### Q43. To reduce reviewer bias, the team wants a second review pass. How should the second pass be set up?
A) Continue in the same context so it remembers its first-pass reasoning
B) Run the second pass in a fresh context (separate agent) so it does not inherit the first pass's conclusions
C) Ask the first pass to rate its own confidence and skip pass two if high
D) Lower temperature on the same context for consistency
Answer: B
Explanation: A genuine second opinion requires fresh context so the reviewer does not inherit and rationalize the first pass's reasoning — directly countering the same-session self-review anti-pattern (A and effectively D). Self-confidence gating (C) is unreliable and can skip review exactly when it is most needed.
Tags: D1, S5, Advanced

### Q44. Claude's review JSON occasionally fails schema validation. What is the robust handling pattern?
A) Accept whatever comes back and hope downstream tolerates it
B) Validate against the schema and, on failure, retry with the validation error fed back to the model
C) Suppress the malformed output and report the PR as passing
D) Strip the invalid fields and merge anyway
Answer: B
Explanation: The reliable pattern is validate-then-retry: check the output against the schema and, on failure, re-prompt including the specific validation error so the model corrects it. Accepting anything (A) and stripping fields to merge (D) propagate bad data. Treating malformed output as a pass (C) is the silent-suppression anti-pattern with real merge risk.
Tags: D4, S5, Advanced

### Q45. What is the safest way to enforce "block merge on any critical-severity finding" in CI?
A) Instruct Claude in the prompt to fail the build on critical findings
B) Have CI code read the structured severity field and fail the job deterministically when any finding is critical
C) Let the model decide whether to fail the build
D) Use the model's self-reported confidence to decide
Answer: B
Explanation: The gating decision must be deterministic CI logic reading the structured severity field, so a critical finding always blocks merge regardless of how the model phrases things. Asking the model to fail the build (A, C) is prompt-enforcement and not guaranteed. Confidence (D) is an unreliable proxy for severity.
Tags: D3, S5, Intermediate

### Q46. The CI review keeps timing out on very large diffs. Best architectural response?
A) Increase the iteration cap and retry forever
B) Chunk the diff into reviewable units, review each (optionally via batch), and aggregate structured findings
C) Skip large diffs and mark them passing
D) Truncate the diff and review only the first part
Answer: B
Explanation: Large inputs should be decomposed into reviewable chunks, each reviewed (batched if appropriate) and the structured findings aggregated — bounding context per unit. Infinite retries (A) waste resources. Skipping large diffs as passing (C) is silent suppression of risk, and truncating (D) leaves most of the change unreviewed.
Tags: D5, S5, Advanced

### Q47. To make per-PR review cheaper, the team sends the same coding-standards guide on every request. What helps most?
A) Prompt caching the stable standards-guide prefix across requests
B) Removing the standards guide entirely
C) Switching to a larger model to read it faster
D) Asking the model to memorize the guide between runs
Answer: A
Explanation: Caching the stable standards-guide prefix lets repeated requests reuse it at lower cost and latency. Removing the guide (B) degrades review quality. A larger model (C) costs more, not less, and models do not persist memory across stateless API calls (D) — caching is the mechanism for prefix reuse.
Tags: D5, S5, Intermediate

### Q48. The team reports "92% of reviews are accurate" but security bugs still slip through. What measurement change exposes the problem?
A) Keep the single aggregate number; 92% is fine
B) Break accuracy down by finding type (security, style, correctness) and track recall on the critical categories
C) Count total findings produced
D) Have the reviewer grade its own accuracy
Answer: B
Explanation: A high aggregate can hide poor recall on a critical slice like security; per-category metrics with attention to recall on high-stakes categories surface it — the aggregate-masking anti-pattern. Total findings (C) measures volume, not correctness, and self-grading (D) inherits the reviewer's bias.
Tags: D4, S5, Advanced

### Q49. For structured review output, which is the more reliable extraction approach?
A) Ask for JSON in prose and parse whatever text appears
B) Use tool_use (or the structured-outputs capability) so the response conforms to a defined schema
C) Have the model wrap JSON in apologies and explanations
D) Rely on the model to never add stray prose
Answer: B
Explanation: Defining a schema via tool_use or structured outputs constrains the response shape, which is far more reliable than parsing free text. Asking for JSON in prose (A, D) frequently yields stray prose or fences that break parsers; wrapping JSON in explanations (C) makes extraction worse. Schema-constrained output is the robust path.
Tags: D4, S5, Intermediate

### Q50. The CI agent should not auto-fix and push; it should only report. How do you guarantee that boundary?
A) Tell it in the prompt not to push
B) Run it with read-only permissions / no write or push tools available in the CI environment
C) Trust it because it usually does not push
D) Set a low iteration cap so it runs out of time before pushing
Answer: B
Explanation: A hard capability boundary comes from the environment: give the CI agent no write/push tools and read-only permissions so pushing is impossible, not merely discouraged. Prompt instructions (A) and trust (C) are the prompt-enforcement anti-pattern. A low iteration cap (D) is an arbitrary backstop that does not actually remove the dangerous capability.
Tags: D3, S5, Advanced

## Scenario 6 — Structured Data Extraction

S6 context: An agent extracts structured records (invoices, contracts, forms) into JSON conforming to a schema. It uses tool_use / structured outputs to constrain shape, validates results, and retries on validation failure. Field-level accuracy and honest handling of missing data matter most.

### Q51. To extract invoice fields into a fixed JSON shape, what is the most reliable mechanism?
A) Ask for JSON in the prompt and parse the text response
B) Define the schema via tool_use (or structured outputs) so the model returns schema-conforming JSON
C) Let the model return prose and extract fields with regex
D) Use extended thinking to produce free-form notes instead of JSON
Answer: B
Explanation: Constraining output with tool_use or structured outputs makes the model return JSON that conforms to your schema, the reliable extraction path. Prompt-asking for JSON (A) and regex over prose (C) are brittle and break on formatting drift. Extended thinking (D) aids reasoning but is not a substitute for schema-constrained output.
Tags: D4, S6, Foundational

### Q52. An extracted record fails JSON-schema validation (wrong type on "total"). What is the correct recovery?
A) Coerce the value silently and store it
B) Retry the extraction, feeding the specific validation error back to the model so it corrects the field
C) Drop the record and report the batch as fully successful
D) Lower the schema's strictness so it always passes
Answer: B
Explanation: Validation-retry feeds the concrete error back so the model fixes the specific field, preserving correctness. Silent coercion (A) can corrupt the value. Dropping the record but reporting full success (C) is silent suppression that hides data loss, and loosening the schema (D) defeats the validation's purpose.
Tags: D4, S6, Intermediate

### Q53. A contract is missing a field the schema marks optional. How should the extractor represent that?
A) Invent a plausible value so the field is populated
B) Return null/omit per the schema and flag the field as not found
C) Copy the value from a different field that looks similar
D) Fail the whole document silently
Answer: B
Explanation: Honest extraction represents genuinely missing data as null/omitted with a not-found flag, never fabricated. Inventing a value (A) and copying a similar field (C) are hallucinations that corrupt the dataset. Silently failing the document (D) is the silent-suppression anti-pattern that hides the gap from downstream consumers.
Tags: D4, S6, Intermediate

### Q54. The extraction job reports 95% accuracy overall, but "handwritten forms" are nearly all wrong. What evaluation fix surfaces this?
A) Keep reporting the single 95% aggregate
B) Report accuracy per document type (typed PDF, scanned, handwritten) so the weak slice is visible
C) Count how many fields were populated
D) Ask the extractor to self-assess accuracy
Answer: B
Explanation: A strong aggregate can hide a failing document type; per-document-type accuracy exposes the handwritten-forms collapse — the aggregate-masking anti-pattern. Counting populated fields (C) rewards fabrication, and self-assessment (D) inherits the model's own bias rather than measuring ground truth.
Tags: D4, S6, Advanced

### Q55. When should the extractor stop retrying a stubborn document and route it elsewhere?
A) When the model's text says "I give up"
B) After a bounded retry budget with persistent validation failures, route to human review with the captured errors
C) Never; keep retrying until it passes
D) When sentiment in the source text turns negative
Answer: B
Explanation: A bounded retry budget that, on persistent failure, routes the document to human review with the captured errors is the sound design — the cap is a backstop and the handoff carries diagnostics. Parsing "I give up" (A) is the natural-language termination anti-pattern. Infinite retries (C) waste cost, and sentiment (D) is irrelevant to extraction validity.
Tags: D5, S6, Advanced

### Q56. The extractor must reject totals that do not equal the sum of line items. Where does this check belong?
A) In the prompt, asking the model to double-check its math
B) In deterministic post-extraction validation code that recomputes and rejects mismatches
C) In a higher max_tokens budget so it computes more carefully
D) In a self-reported confidence threshold
Answer: B
Explanation: Arithmetic invariants belong in deterministic validation that recomputes the sum and rejects mismatches, independent of the model. Asking the model to check its math (A) is prompt-enforcement with no guarantee. Token budget (C) does not enforce correctness, and confidence (D) is an unreliable proxy for arithmetic validity.
Tags: D4, S6, Advanced

### Q57. To extract from 50,000 documents overnight with no latency requirement, which approach is most cost-effective?
A) Synchronous one-at-a-time calls during business hours
B) The Batch API to process the documents asynchronously at lower cost
C) A single prompt containing all 50,000 documents
D) Manual extraction by the team
Answer: B
Explanation: The Batch API processes large, latency-tolerant workloads asynchronously at reduced cost — ideal for an overnight 50k-document job. Synchronous calls (A) are slow and pricier at scale. One mega-prompt (C) cannot fit and would be unreliable, and manual extraction (D) defeats automation.
Tags: D5, S6, Intermediate

### Q58. The same long extraction instructions and schema are sent on every document. What reduces repeated cost?
A) Prompt caching the stable instruction/schema prefix
B) Shortening the schema so it omits fields you need
C) Switching to a smaller model that ignores the schema
D) Removing validation to save tokens
Answer: A
Explanation: Caching the stable instruction-and-schema prefix lets each document request reuse it at lower cost and latency. Trimming needed fields (B) loses data, a model that ignores the schema (C) breaks extraction, and removing validation (D) trades correctness for marginal savings — the opposite of what this scenario values.
Tags: D5, S6, Intermediate

### Q59. The model sometimes adds an explanatory sentence before the JSON, breaking the parser. Best fix?
A) Add "do not include any prose" to the prompt and hope it complies
B) Use tool_use / structured outputs so the response is the structured object, not free text
C) Strip everything before the first "{" with a regex and move on
D) Increase temperature so it varies the wording
Answer: B
Explanation: Using tool_use or structured outputs makes the response a structured object with no surrounding prose to strip, eliminating the failure mode at the source. Prompt pleading (A) is unreliable. Regex stripping (C) is a fragile workaround that hides the real cause, and raising temperature (D) makes stray prose more likely, not less.
Tags: D4, S6, Intermediate

### Q60. A reviewer wants to trust the extraction pipeline's quality before production. Which validation strategy is strongest?
A) Trust the model's self-reported per-field confidence
B) Evaluate field-level accuracy against a labeled gold set, sliced by document type and field
C) Spot-check a handful of documents the engineer chooses
D) Count total extractions completed without error messages
Answer: B
Explanation: Field-level accuracy against a labeled gold set, sliced by document type and field, gives an honest, actionable quality picture and exposes weak slices. Self-reported confidence (A) is the confidence-as-signal anti-pattern. Convenience spot-checks (C) are not representative, and a no-error count (D) measures throughput, not correctness.
Tags: D4, S6, Advanced

### Q61. The schema requires an ISO-8601 date but source documents use many formats. Where should normalization happen?
A) Ask the model in the prompt to "always output ISO-8601" and trust it
B) Have the model extract the date, then normalize/validate to ISO-8601 in deterministic post-processing
C) Accept any date string the model emits
D) Use sentiment to decide whether the date is trustworthy
Answer: B
Explanation: The reliable design extracts the raw date and then normalizes and validates it to ISO-8601 in deterministic code, so format guarantees do not depend on model compliance. Prompt-only formatting (A) is the enforcement anti-pattern. Accepting any string (C) breaks the schema contract, and sentiment (D) is irrelevant to date validity.
Tags: D4, S6, Advanced

### Q62. Across S1–S6, which single principle most reliably prevents agents from acting on false premises?
A) Higher iteration caps so the agent has more chances
B) Surfacing real errors and missing data honestly (structured errors, explicit not-found) instead of suppressing them
C) Self-reported confidence scores at every step
D) Longer system prompts begging for correctness
Answer: B
Explanation: Honest error and missing-data handling — structured errors and explicit not-found rather than empty-as-success — is the cross-cutting reliability principle that keeps every scenario's agent from building on false premises. Iteration caps (A) are only a backstop, confidence scores (C) are an unreliable signal, and longer pleading prompts (D) are the prompt-enforcement anti-pattern.
Tags: D5, S1, Advanced
