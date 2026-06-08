# Domain D1 — Advanced & Edge-Case Questions

Advanced and edge-case practice items for Domain 1 (Agentic Architecture & Orchestration, 27% of the CCA-F exam). These questions push past the foundational bank into the harder, scenario-anchored material that the existing D1 set under-covers: the **full `stop_reason` value set** (`pause_turn`, `refusal`, `model_context_window_exceeded`), **server-tool sampling loops**, **empty-response and truncated-tool-use edge cases**, **retryable-vs-terminal error classification**, **partial-failure synthesis**, **subagent fan-out economics**, **prompt caching interactions in the agent loop**, and **layered termination logic**. Every fact here is grounded in current Anthropic platform behavior (platform.claude.com/docs — "Handling stop reasons," "Batch processing," "Building effective agents," and "How we built our multi-agent research system").

Note on explanations: distractors are referenced by their content (not by option letter) so the rationale stays correct regardless of answer position.

## The Full stop_reason Value Set and Server-Tool Edge Cases

### Q1. An S1 support agent uses the server-side web_search tool to research a policy. The API returns a response whose stop_reason is "pause_turn" and whose last content block is a server_tool_use with no matching result. What is the correct loop behavior?

- **A.** Treat pause_turn as a terminal error and escalate the ticket to a human
- **B.** Strip the incomplete server_tool_use block and re-prompt from scratch
- **C.** Append the assistant response as-is to the messages array and call the API again so Claude can continue its server-tool work
- **D.** Parse the partial text for a completion phrase to decide whether to stop

**Answer:** `C`

**Explanation:** pause_turn is returned when the server-side sampling loop for a server tool (web_search/web_fetch) reaches its iteration limit (default 10) mid-turn; the documented handling is to send the assistant's response back unmodified so Claude resumes and finishes. Treating it as terminal abandons recoverable work, and stripping the block or re-prompting from scratch discards progress and breaks the alternating-role structure. Parsing prose for completion is anti-pattern #1.

**Tags:** `Domain 1` `Advanced` `S1` `Agent Loop` `stop_reason`

### Q2. Which stop_reason indicates the model declined to answer for safety reasons, and on Claude Opus 4.7+ carries a stop_details object naming the policy category?

- **A.** end_turn
- **B.** refusal
- **C.** max_tokens
- **D.** pause_turn

**Answer:** `B`

**Explanation:** refusal means Claude declined to generate on safety grounds; on Opus 4.7 and later it includes a stop_details object whose category field (e.g., "cyber" or "bio", or null) lets you branch deterministically without parsing the explanation text. end_turn is a clean natural finish, max_tokens is output truncation, and pause_turn is a server-tool sampling-loop pause — none of these is a safety refusal. A robust agent loop must branch on refusal explicitly rather than treat it as a normal end_turn.

**Tags:** `Domain 1` `Advanced` `Agent Loop` `stop_reason` `Reliability`

### Q3. An agent loop checks only for tool_use and end_turn, treating every other stop_reason as "done." A response comes back with stop_reason "refusal" and empty content, so the agent reports success to the user. What is the architectural flaw?

- **A.** The loop should have used a higher iteration cap
- **B.** Nothing is wrong; refusal is equivalent to end_turn
- **C.** The agent should parse the empty content for the word "refuse"
- **D.** The loop's stop_reason handling is incomplete: refusal (and pause_turn, max_tokens, model_context_window_exceeded) must each be handled distinctly, not collapsed into the end_turn success path

**Answer:** `D`

**Explanation:** Robust agents branch on the full stop_reason set; silently mapping refusal/max_tokens/model_context_window_exceeded onto the success path reports failure as success (a cousin of anti-pattern #7). refusal is a safety decline, not a completed task, and needs its own handling (surface, rephrase, or escalate). Raising the cap is irrelevant, equating refusal with end_turn is the bug itself, and parsing content for "refuse" is brittle natural-language control (anti-pattern #1) when the deterministic field is right there.

**Tags:** `Domain 1` `Advanced` `Agent Loop` `stop_reason` `Reliability`

### Q4. During an agentic task, a response returns stop_reason "max_tokens" and its final content block is an incomplete tool_use block. What is the correct recovery?

- **A.** Retry the request with a higher max_tokens so Claude can emit the complete tool_use block, then execute it
- **B.** Execute the partial tool call with whatever arguments are present
- **C.** Treat it as end_turn and return the truncated text to the user
- **D.** Discard the conversation and start a new session

**Answer:** `A`

**Explanation:** When max_tokens truncates a response mid tool_use, the tool input JSON is incomplete and unsafe to execute; the documented fix is to retry with a higher max_tokens to obtain the full block. Executing partial/garbled arguments risks malformed or dangerous tool calls, treating truncation as end_turn silently corrupts the run, and discarding the session throws away recoverable context. Detecting "stop_reason == max_tokens AND last block is tool_use" is the precise trigger.

**Tags:** `Domain 1` `Advanced` `Agent Loop` `stop_reason` `Reliability`

### Q5. A long-context S4 codebase-exploration agent requests a large max_tokens. The response returns stop_reason "model_context_window_exceeded" rather than "max_tokens." What does this signal and how should you treat the partial output?

- **A.** An HTTP error occurred and the request failed
- **B.** The model refused for safety reasons
- **C.** The batch expired after 24 hours
- **D.** Generation stopped because the model's total context window limit was reached before the max_tokens limit; the returned content is still valid but was bounded by context, so handle it as a (possibly) truncated success

**Answer:** `D`

**Explanation:** model_context_window_exceeded is a successful response stop_reason (not an error) that fires when output generation hits the context-window ceiling before reaching max_tokens, letting you request a large max_tokens without precomputing input size; the content is valid but may be cut off and should be handled like a truncation. It is not an HTTP failure, not a safety refusal, and unrelated to batch expiry. On Sonnet 4.5+ it is on by default; older models need the model-context-window-exceeded beta header.

**Tags:** `Domain 1` `Advanced` `S4` `stop_reason` `Context Management`

### Q6. After returning a tool_result, an agent receives a response with stop_reason "end_turn" but empty content (2-3 tokens, no text). The team blames the model. What is the most likely cause and fix?

- **A.** The model is broken; switch providers
- **B.** Text blocks were added in the same user turn right after the tool_result, teaching Claude to expect user input after every tool use; remove the trailing text and send the tool_result alone, using a fresh "please continue" user message only as a last resort
- **C.** The iteration cap was too low
- **D.** stop_reason should have been parsed from the response text

**Answer:** `B`

**Explanation:** Empty end_turn responses after tool results are a documented edge case caused by appending a text block immediately after the tool_result (Claude learns the pattern "user always speaks after tool results" and ends its turn) or by re-sending an already-complete response. The fix is to send tool_result content with no trailing text, and only add a new "Please continue" user message if the empty response persists. It is not a model defect, not an iteration-cap issue, and stop_reason is a structured field, never parsed from prose.

**Tags:** `Domain 1` `Advanced` `Agent Loop` `Edge Case` `Reliability`

### Q7. Why is honoring stop_reason "pause_turn" specifically important when an agent uses server tools rather than client-side tools?

- **A.** Server tools never return tool_use, so pause_turn is the only signal you get
- **B.** pause_turn only occurs in batch mode
- **C.** The server runs its own multi-step sampling loop for tools like web_search; when that internal loop hits its iteration limit mid-turn, the API pauses and expects you to continue the conversation, otherwise the server-tool work is left unfinished
- **D.** pause_turn means the rate limit was exceeded and you must back off

**Answer:** `C`

**Explanation:** Server tools execute inside an Anthropic-managed sampling loop; pause_turn surfaces when that loop reaches its iteration limit (default 10) before completing, and you resume by re-sending the response so Claude continues. Client-side tools surface as tool_use, which you execute yourself, so the signals are distinct. pause_turn is not batch-only and is not a rate-limit signal (that would be an HTTP 429 error, not a stop_reason). Any agent loop using server tools must handle pause_turn.

**Tags:** `Domain 1` `Advanced` `S1` `Agent Loop` `stop_reason`

### Q8. An agent must distinguish a successful-but-stopped response from a failed request. Which statement is correct about stop_reason versus HTTP errors?

- **A.** stop_reason and HTTP status codes are the same thing
- **B.** stop_reason values (end_turn, tool_use, max_tokens, pause_turn, refusal, etc.) appear in the body of a successful 200 response and explain why generation stopped, whereas HTTP 4xx/5xx errors (429, 500, 529) indicate the request failed to process at all
- **C.** A refusal stop_reason is returned as an HTTP 403
- **D.** max_tokens is returned as an HTTP 413

**Answer:** `B`

**Explanation:** stop_reason is part of a successful response body and tells you why generation ended normally; transport-level failures are signaled by HTTP status codes and exceptions instead. Refusal is a stop_reason on a 200 response (not a 403), and truncation is a stop_reason (not a 413). Conflating the two leads to mis-handling: you retry/back off on 429/500/529 errors, but you branch on stop_reason for in-band outcomes.

**Tags:** `Domain 1` `Advanced` `Agent Loop` `Error Handling`

## Retryable vs. Terminal Error Classification

### Q9. An S1 tool layer retries every failed API call with exponential backoff, including HTTP 400 (invalid request) and 401 (auth) errors. Why is this wrong?

- **A.** 400 and 401 are non-transient client errors (bad request, bad credentials) that will fail identically on every retry; retrying wastes time and budget and can trip rate limits — only retry transient errors like 429, 500, 503, and 529
- **B.** It is correct; all errors are transient
- **C.** The fix is to retry 400/401 even faster
- **D.** 401 should be retried but 400 should crash the process

**Answer:** `A`

**Explanation:** Retry-with-backoff is for transient/retryable failures (429 rate limit, 500/503 server errors, 529 overloaded, network timeouts); deterministic client errors like 400 (malformed request) and 401 (authentication) will not succeed on retry and must be surfaced and fixed, not looped. Retrying them faster compounds the waste, and there is no basis for treating 401 as retryable while 400 crashes. Correct fault tolerance starts with classifying retryable vs. terminal.

**Tags:** `Domain 1` `Advanced` `S1` `Fault Tolerance` `Error Handling`

### Q10. An Anthropic API call returns HTTP 529 ("overloaded"). How should a production agent treat this versus a 429?

- **A.** 529 is terminal; abort the whole agent
- **B.** 529 means the prompt was rejected for safety
- **C.** 529 should be retried instantly in a tight loop
- **D.** Both 529 (service overloaded) and 429 (rate limited) are transient and retryable with exponential backoff and jitter; honor any Retry-After guidance and retry within a bounded budget before escalating

**Answer:** `D`

**Explanation:** 529 indicates the service is temporarily overloaded and 429 indicates you exceeded a rate limit; both are transient, so back off exponentially with jitter and retry within a bounded budget, respecting Retry-After when present. Aborting the agent on a transient hiccup is brittle, 529 is not a safety rejection (that surfaces as a refusal stop_reason on a 200), and tight instant retries worsen the overload (thundering herd). Distinguish transient (retry) from terminal (surface) is the core skill.

**Tags:** `Domain 1` `Advanced` `Fault Tolerance` `Error Handling`

### Q11. A mutating tool (issue_refund) times out with no response. The agent retries and the refund is applied twice. What design property was missing and what is the fix?

- **A.** The tool needed a longer timeout only
- **B.** Idempotency: because a timeout does not prove the original write failed server-side, retries of mutating operations must carry an idempotency key (or be naturally idempotent) so a duplicate request is deduplicated rather than re-applied
- **C.** The agent should never retry; one attempt only
- **D.** The fix is to raise the iteration cap so it has more attempts

**Answer:** `B`

**Explanation:** A timeout is ambiguous — the refund may have committed before the response was lost — so blindly retrying a non-idempotent write double-applies it; the fix is an idempotency key so the server deduplicates the retried call. A longer timeout alone does not solve the ambiguity, banning retries removes fault tolerance, and a higher iteration cap is unrelated and would only increase the chance of duplication. Read tools are naturally safe; writes are the dangerous case.

**Tags:** `Domain 1` `Advanced` `S1` `Fault Tolerance`

### Q12. Which retry policy is correct for a flaky upstream MCP tool that occasionally returns transient 503s during a 40-minute research run?

- **A.** Exponential backoff with jitter, capped at a bounded number of attempts; if still failing, return a structured, diagnostic error so the model can re-scope or escalate, and preserve checkpointed state so the run can resume
- **B.** Infinite immediate retries until it succeeds
- **C.** Swallow the 503 and return an empty result so the loop keeps moving
- **D.** Abort and discard all prior work from the 40-minute run

**Answer:** `A`

**Explanation:** Bounded exponential backoff with jitter handles transient 503s without hammering the service, and a clear structured error on persistent failure lets the model adapt instead of guessing; durable checkpoints prevent losing 40 minutes of work. Infinite immediate retries cause retry storms and unbounded cost, swallowing the error and returning empty is anti-pattern #7, and aborting the whole run throws away recoverable progress.

**Tags:** `Domain 1` `Advanced` `S3` `Fault Tolerance` `State`

## Partial Failure, Synthesis, and Per-Slice Evaluation

### Q13. In S3, four of five subagents return valid structured results; one fails permanently after retries. What is the most defensible coordinator behavior?

- **A.** Report 100% completeness using only the four successful results
- **B.** Discard all five results and abort the mission
- **C.** Synthesize from the four successful results while explicitly flagging the missing slice (its scope and that it failed) in the final output, so the gap is visible rather than hidden
- **D.** Fabricate plausible content for the failed slice to fill the gap

**Answer:** `C`

**Explanation:** Graceful degradation means isolating the failure to one spoke and still synthesizing, but the synthesis must surface the known gap — silently reporting 100% completeness masks a failed slice (anti-pattern #7 / #10) and misleads the consumer. Discarding everything is brittle overreaction, and fabricating content for the missing slice is a fabrication failure worse than an honest gap. Visible, attributed gaps preserve trust and auditability.

**Tags:** `Domain 1` `Advanced` `S3` `Fault Tolerance` `Reliability`

### Q14. A multi-agent extraction pipeline (S6) reports 94% aggregate accuracy and ships. Later, invoices fail at 55% while receipts pass at 99%. What practice would have caught this earlier?

- **A.** Trusting the single aggregate number, which is sufficient
- **B.** Raising the iteration cap on each worker
- **C.** Adding more subagents to amortize the error
- **D.** Per-slice (per-document-type) evaluation, because an aggregate average can mask a category performing far below threshold while high-volume easy categories inflate the mean

**Answer:** `D`

**Explanation:** Aggregate accuracy hides per-category failures — anti-pattern #10 — so a strong receipts class can mask a broken invoices class until production; evaluating per document type surfaces the weak slice for targeted fixes. The aggregate number is precisely the trap, and neither a higher iteration cap nor more subagents addresses a per-type accuracy defect. Slice your metrics by the dimensions that actually vary in difficulty.

**Tags:** `Domain 1` `Advanced` `S6` `Reliability Evaluation`

### Q15. A coordinator determines a subagent "finished" by reading its returned prose for an upbeat tone ("I found great results!"). The subagent actually returned no usable data. What two anti-patterns combine here?

- **A.** Natural-language termination (#1) plus silent error suppression (#7), since prose tone is treated as a completion signal and an empty/failed result is accepted as success
- **B.** Sentiment escalation (#5) plus self-confidence (#4)
- **C.** Iteration cap (#2) plus too many tools (#8)
- **D.** Same-session self-review (#9) plus aggregate metrics (#10)

**Answer:** `A`

**Explanation:** Inferring completion from prose tone is anti-pattern #1 (the deterministic signal is a valid, contract-conforming structured result), and accepting a result that contains no usable data as success is anti-pattern #7 (silent suppression). The fix is to verify the subagent returned a schema-valid result with actual content before counting it complete. The other pairs describe unrelated failures.

**Tags:** `Domain 1` `Advanced` `S3` `Termination` `Error Handling`

## Subagent Fan-Out Economics and Isolation Edge Cases

### Q16. Anthropic's multi-agent research write-up notes that subagents multiply token usage. Roughly how should an architect reason about when the multi-agent token premium is justified?

- **A.** Multi-agent always pays for itself, so use it everywhere
- **B.** Token cost is irrelevant once subagents are involved
- **C.** The premium is justified when the task is high-value and genuinely benefits from parallel breadth (many independent sub-investigations); for narrow, sequential, or low-value tasks the single-agent design is cheaper and simpler
- **D.** Justify multi-agent only when you have exactly 18 tools to distribute

**Answer:** `C`

**Explanation:** Multi-agent systems trade substantially higher token spend (Anthropic reported multi-agent research uses far more tokens than single-agent chat) for breadth and parallelism, so the premium is worth it on valuable, breadth-heavy tasks and wasteful on narrow or low-value ones. It does not always pay for itself, token cost is never irrelevant, and tool count is unrelated to the breadth/value calculus. Match architecture complexity to task value.

**Tags:** `Domain 1` `Advanced` `S3` `Cost/Latency Trade-offs`

### Q17. A coordinator passes its entire 60k-token context to each of eight subagents "so they have full information." What is the most likely consequence?

- **A.** Context bloat and distraction: each subagent inherits irrelevant clutter, token cost multiplies across all eight, and reasoning quality drops; the fix is a concise per-subagent briefing (relevant facts + precise objective) within an isolated context
- **B.** Subagents become more accurate because more context is always better
- **C.** The subagents will run strictly faster
- **D.** stop_reason handling will break

**Answer:** `A`

**Explanation:** Dumping the full coordinator context into every subagent reintroduces the clutter that context isolation exists to prevent, multiplies token cost by the fan-out factor, and degrades focus — the opposite of the intended benefit. The right move is a tailored, minimal briefing per subagent. More context is not always better, it does not make subagents faster, and it is unrelated to stop_reason.

**Tags:** `Domain 1` `Advanced` `S3` `Subagent Isolation` `Cost/Latency Trade-offs`

### Q18. Why does Anthropic favor returning a condensed, structured result from each subagent to the coordinator rather than the subagent's full transcript?

- **A.** Transcripts cannot be serialized
- **B.** Full transcripts always use fewer tokens than summaries
- **C.** The coordinator cannot read structured data
- **D.** The subagent's raw transcript (file reads, dead-ends, intermediate searches) would flood the coordinator's context, exhaust its window as fan-out grows, and inject noise; a condensed structured result preserves the findings while keeping the coordinator's context clean and bounded

**Answer:** `D`

**Explanation:** The whole point of subagent isolation is that workers digest their exploration and hand back only distilled findings, so the coordinator's context stays clean and does not blow up as the number of subagents grows. Transcripts can be serialized, summaries are smaller than raw dumps (not larger), and coordinators of course read structured data. This is the multi-agent analogue of single-agent compaction.

**Tags:** `Domain 1` `Advanced` `S3` `Subagent Isolation` `Context Management`

### Q19. An orchestrator spawns one subagent per word in the user's query to "maximize parallelism." Why is this wrong, and what is the right heuristic?

- **A.** It is right; more subagents always means more breadth
- **B.** The right count is always exactly five
- **C.** Subagent count should scale to the query's actual breadth and complexity, not to a meaningless proxy like word count; over-spawning multiplies token cost and coordination overhead with diminishing returns and can fragment a coherent subtask into nonsense slices
- **D.** The orchestrator should spawn zero subagents and do everything itself

**Answer:** `C`

**Explanation:** Effective orchestrators size the fan-out to the real structure of the problem (how many genuinely independent sub-investigations exist), because each subagent adds token and coordination cost; word count is an absurd proxy that shatters the task and explodes spend. There is no universal magic number, and an orchestrator doing everything itself defeats hub-and-spoke (the "orchestrator executes everything" anti-pattern). Right-size the decomposition to the query.

**Tags:** `Domain 1` `Advanced` `S3` `Orchestrator-Workers` `Cost/Latency Trade-offs`

### Q20. Two subagents must both update a shared findings document concurrently. Which design avoids corruption while preserving parallelism?

- **A.** Partition writes (each subagent owns a disjoint section), use append-only logs, or have the coordinator own the final merge — so concurrent work proceeds without race conditions
- **B.** Let both write to the same mutable cells with no coordination
- **C.** Force the subagents to run sequentially, eliminating parallelism entirely
- **D.** Disable stop_reason so writes cannot collide

**Answer:** `A`

**Explanation:** Uncoordinated concurrent writes to shared mutable state cause race conditions and inconsistent results; partitioning, append-only logs, or a coordinator-owned merge let subagents keep running in parallel while staying consistent. Unguarded shared writes are the bug, serializing everything needlessly discards the latency benefit, and stop_reason has nothing to do with write coordination. Preserve parallelism with safe write patterns.

**Tags:** `Domain 1` `Advanced` `S3` `Subagent Isolation` `Reliability`

## Layered Termination and Loop-Control Edge Cases

### Q21. A robustly designed agent loop is observed hitting its max_iterations backstop on roughly 30% of runs to terminate. What does this indicate?

- **A.** The system is healthy; the cap is the intended stop mechanism
- **B.** The cap should be raised to 100 so it stops hitting it
- **C.** The fix is to parse the model's text for "done" instead
- **D.** A design defect: the cap is a safety backstop, not the control mechanism, so frequent cap-hits mean the agent is not reaching a natural end_turn — investigate looping, poor tool results, or an unbounded task, and treat cap-hits as anomalies to log/escalate

**Answer:** `D`

**Explanation:** Relying on the iteration cap to stop work is anti-pattern #2; the cap exists only to bound runaway loops, so hitting it 30% of the time signals the agent rarely reaches a clean end_turn and the underlying loop or task design is broken. Raising the cap masks the symptom without fixing the cause, and switching to prose parsing swaps in anti-pattern #1. A healthy agent terminates on end_turn and hits the cap only as a rare anomaly.

**Tags:** `Domain 1` `Advanced` `Termination Anti-Pattern` `Reliability`

### Q22. Which termination design for an S3 subagent is the most robust, combining model intent with deterministic verification?

- **A.** Stop after exactly 10 tool calls regardless of findings
- **B.** Stop the moment the subagent's text says "I think that's enough"
- **C.** Let the subagent reach end_turn after emitting its required schema-conforming structured result, bound the run with a safety iteration backstop, and have the coordinator verify the result is valid and complete before counting the subtask done
- **D.** Keep searching until the context window overflows

**Answer:** `C`

**Explanation:** The strongest design layers three things: the model's own end_turn (after delivering the contract-required structured output), a safety backstop cap, and a deterministic coordinator-side completeness/validity check — so completion is both model-driven and verified. A fixed call count is anti-pattern #2, a prose-based stop is anti-pattern #1, and running until overflow is reckless and corrupts context. Combine intent with verification.

**Tags:** `Domain 1` `Advanced` `S3` `Termination`

### Q23. An agent's control flow is implemented entirely in the system prompt: "Call tools in order, then stop when the task is done." Which critique is most accurate?

- **A.** It conflates prompting with control: deterministic tool ordering and termination must come from loop logic that branches on stop_reason (continue on tool_use, stop on end_turn) and code-level orchestration; trusting the prompt to self-order and self-stop stacks anti-patterns #1 and #3
- **B.** This is production-grade control
- **C.** It only fails because the prompt is too short
- **D.** It is fine at temperature 0

**Answer:** `A`

**Explanation:** Tool ordering and termination are control concerns that belong in deterministic code keyed on stop_reason, not in natural-language pleas; relying on the prompt to guarantee them combines natural-language termination (#1) and prompt-based enforcement of critical control flow (#3). Prompt length does not convert guidance into a guarantee, and temperature 0 reduces variance but still cannot make prompting deterministic control.

**Tags:** `Domain 1` `Advanced` `Agent Loop` `Deterministic Enforcement`

### Q24. In a tool-use loop, after stop_reason "tool_use" you execute the tool and append results. What is the precise message structure that keeps the loop valid?

- **A.** Append a single user message containing the tool_result block(s), each keyed by the matching tool_use_id, after first appending the assistant's tool_use turn
- **B.** Append only the tool_result with no tool_use_id and skip the assistant turn
- **C.** Replace the whole history with just the latest tool output
- **D.** Add a text block right after the tool_result to narrate progress

**Answer:** `A`

**Explanation:** The valid structure preserves the assistant's tool_use turn, then a user turn whose tool_result block(s) reference the corresponding tool_use_id so Claude can match output to request. Omitting the tool_use_id breaks the linkage, wiping history loses context, and adding a text block immediately after the tool_result is a documented cause of empty end_turn responses (it trains Claude to expect user input after every tool use). Keep tool_result turns clean and correctly keyed.

**Tags:** `Domain 1` `Advanced` `Agent Loop` `Edge Case`

### Q25. A streaming agent reads stop_reason from each event. In which event is stop_reason actually populated?

- **A.** The message_start event
- **B.** Every content_block_delta event
- **C.** The message_delta event (it is null at message_start and not present in other event types)
- **D.** Only after the stream closes with no event

**Answer:** `C`

**Explanation:** In streaming, stop_reason is null in the initial message_start event and is delivered in the message_delta event near the end of the stream; it is not present in other event types. Reading it from message_start yields null, and content_block_delta events carry incremental content, not the stop reason. Knowing where the field lands matters for correctly driving an agent loop over a stream.

**Tags:** `Domain 1` `Advanced` `Agent Loop` `Streaming`

## Batch, Caching, and Architecture-Selection Edge Cases

### Q26. An S5 CI/CD pipeline must run 50,000 independent code-review classifications overnight at lowest cost, with no latency requirement. Which approach and which concrete trade-off are correct?

- **A.** Synchronous Messages API at high concurrency for the fastest possible turnaround
- **B.** A hub-and-spoke multi-agent system, one coordinator per classification
- **C.** Extended thinking at maximum budget on every classification for accuracy
- **D.** The Message Batches API, which processes requests asynchronously at roughly 50% lower cost, with most batches finishing within an hour but a hard 24-hour expiration window after which unprocessed requests expire unbilled

**Answer:** `D`

**Explanation:** For large volumes of independent, non-time-sensitive requests, the Batches API is purpose-built: asynchronous processing at about 50% lower cost, most batches done within an hour, results available when all complete or after the 24-hour expiration (whichever comes first). Synchronous high-concurrency costs more and adds orchestration burden, a multi-agent system per classification is gross over-engineering, and max extended-thinking on trivial classifications wastes budget. Know the 50% / 24-hour figures.

**Tags:** `Domain 1` `Advanced` `S5` `Cost/Latency Trade-offs` `Batch API`

### Q27. A team plans to use the Batch API but wants to pre-warm the prompt cache by submitting requests with max_tokens 0 inside the batch. Why does this fail?

- **A.** Each batched request must have max_tokens of at least 1; max_tokens 0 (cache pre-warming) is unsupported in a batch because an ephemeral cache entry written during async batch processing would likely expire before any follow-up request runs
- **B.** Batch requests cannot use prompt caching at all
- **C.** max_tokens 0 is supported but triple-billed in batches
- **D.** The batch would succeed but silently drop all results

**Answer:** `A`

**Explanation:** Batched requests require max_tokens >= 1; the max_tokens: 0 cache pre-warming trick is not supported inside a batch because the asynchronous timing means the short-lived cache entry would likely expire before a dependent request executes. Prompt caching itself still works within batched requests (so "cannot use caching at all" is wrong), there is no triple-billing rule, and the batch does not silently drop results. This is a real edge-case constraint.

**Tags:** `Domain 1` `Advanced` `S5` `Batch API` `Prompt Caching`

### Q28. In a long agent loop, the system prompt and tool definitions are large and stable across turns. How does prompt caching interact with the loop, and what is the failure mode if the stable prefix changes every turn?

- **A.** Caching has no effect in agent loops
- **B.** Caching only helps embeddings, not agent loops
- **C.** A large stable prefix (system prompt + tool schemas) cached once yields cache hits on subsequent turns, cutting input cost and time-to-first-token; if you mutate that prefix every turn (e.g., reordering tools or rewriting the system prompt), you invalidate the cache and lose the savings
- **D.** Caching forces every turn to re-read the full context at full price

**Answer:** `C`

**Explanation:** Prompt caching pays off precisely in agent loops where a large prefix repeats across many turns, producing cache hits that reduce cost and latency; the failure mode is invalidating the cache by changing the cached prefix each turn, which forces full reprocessing. Caching is not limited to embeddings, it does help agent loops, and a stable cached prefix avoids re-reading at full price (that is the whole point). Keep the cached prefix stable.

**Tags:** `Domain 1` `Advanced` `Cost/Latency Trade-offs` `Prompt Caching`

### Q29. A scenario requires reformatting one small JSON object exactly the same way every time, with no decision-making. An engineer proposes a hub-and-spoke multi-agent system "for robustness." What is the correct critique using the complexity ladder?

- **A.** Multi-agent is the safest default, so this is fine
- **B.** The fix is to add an evaluator-optimizer loop on top of the multi-agent system
- **C.** Use orchestrator-workers so the orchestrator can decide how to reformat
- **D.** This violates simplicity-first: a deterministic single LLM call (or even pure code) handles a fixed, decision-free transformation; climbing to multi-agent adds token cost, latency, and nondeterminism with zero benefit

**Answer:** `D`

**Explanation:** The complexity ladder runs single call -> workflow -> single agent -> multi-agent, and you climb only when the task demands it; a fixed, decision-free reformat sits at the bottom rung, so multi-agent is pure over-engineering that adds cost, latency, and nondeterminism. Layering an evaluator-optimizer or orchestrator-workers on top compounds the over-engineering for a task with no decisions to make. Match the architecture to the task's actual complexity.

**Tags:** `Domain 1` `Advanced` `Architecture Selection` `S6`

### Q30. An orchestrator-workers design dynamically decides subtasks at runtime, but a teammate insists it is "the same as parallelization with a fixed fan-out." What is the precise distinction, and when does each apply?

- **A.** Parallelization splits a predefined, known-in-advance set of independent subtasks; orchestrator-workers has the model determine the subtasks at runtime from the specific input — so use parallelization when you can enumerate the slices up front and orchestrator-workers when you cannot
- **B.** They are identical
- **C.** Orchestrator-workers cannot run subtasks concurrently
- **D.** Parallelization requires an LLM orchestrator while orchestrator-workers does not

**Answer:** `A`

**Explanation:** The defining difference is dynamism: parallelization (sectioning/voting) operates on a fixed set of subtasks known before runtime, while orchestrator-workers lets the orchestrator decide the subtasks based on the input it sees — which is why orchestrator-workers fits open-ended work like research or a code change touching an unknown set of files. Orchestrator-workers can absolutely run workers concurrently, and the LLM-orchestrator framing is backwards. Enumerable up front -> parallelization; unknown until runtime -> orchestrator-workers.

**Tags:** `Domain 1` `Advanced` `Orchestrator-Workers` `Workflow Patterns`

### Q31. A breadth-first S3 research query is decomposed into eight independent sub-investigations. The team reports total latency equals the sum of all eight subagents' runtimes. What architectural mistake is implied, and what is the fix?

- **A.** Nothing; summed latency is expected for any multi-agent design
- **B.** The fix is to add an evaluator-optimizer loop to speed them up
- **C.** The subagents are being run sequentially despite being independent; fanning them out to run concurrently collapses total latency toward the slowest single branch instead of the sum, which is the latency benefit of parallel subagents
- **D.** The fix is to merge all eight into one mega-agent context

**Answer:** `C`

**Explanation:** Independent subtasks should run concurrently so wall-clock latency approaches the slowest branch, not the sum; summed latency reveals they are being executed serially, forfeiting parallelism's main payoff. An evaluator-optimizer loop is a quality mechanism, not a concurrency one, and merging into a single mega-agent context serializes the work and risks the too-many-tools / context-pollution anti-patterns. Run independent branches in parallel.

**Tags:** `Domain 1` `Advanced` `S3` `Cost/Latency Trade-offs`

### Q32. An evaluator-optimizer loop in S5 uses the same agent in the same context to both author and review the code. It rarely finds defects. What is the root cause and the fix?

- **A.** The evaluator needs a higher temperature
- **B.** The fix is to skip review and trust the author
- **C.** The fix is to give the author more tools
- **D.** Same-session self-review (anti-pattern #9): the reviewer inherits the author's reasoning chain, assumptions, and blind spots, so it rationalizes rather than catches errors; run the evaluator in a fresh context or as a separate agent with independent reasoning

**Answer:** `D`

**Explanation:** When the evaluator shares the author's context, it inherits the same assumptions and tends to confirm rather than challenge — anti-pattern #9 — which is why it rarely surfaces defects. Decoupling the reviewer into a fresh context or a distinct agent restores independent judgment. Raising temperature does not remove shared bias, skipping review abandons quality control, and more tools for the author does not address the review's lack of independence.

**Tags:** `Domain 1` `Advanced` `S5` `Self-Review Anti-Pattern`

### Q33. A scenario combines a router (billing vs. technical vs. account), and within the technical path a generate->schema-validate->repair loop. Which composite description is correct, and is composing patterns acceptable?

- **A.** This composes routing (classify-then-dispatch) with an evaluator-optimizer loop whose evaluator is a deterministic schema validator; composing workflow patterns to match task structure is explicitly encouraged, not forbidden
- **B.** Patterns cannot be composed; pick exactly one
- **C.** This is pure orchestrator-workers because a router decides things
- **D.** This is parallelization because there are multiple paths

**Answer:** `A`

**Explanation:** Routing classifies and dispatches the input, and the technical path's generate->validate->repair cycle is an evaluator-optimizer loop with a code-based (schema) evaluator; Anthropic treats the five patterns as composable building blocks chosen by task structure. A router is not orchestrator-workers (its categories are predefined, not dynamically derived), and multiple mutually exclusive paths via classification is routing, not parallel execution of independent slices.

**Tags:** `Domain 1` `Advanced` `Workflow Patterns` `S6`

### Q34. Why is a deterministic schema validator often a stronger "evaluator" than an LLM critic in an extraction-repair loop (S6)?

- **A.** An LLM critic is always cheaper
- **B.** Schema validators can repair semantic meaning, which LLMs cannot
- **C.** For objective, well-specified constraints (types, required fields, enums, formats), a code validator gives a deterministic, repeatable pass/fail with no variance or self-bias, whereas an LLM critic can be inconsistent and may miss or hallucinate violations
- **D.** LLM critics cannot be used in loops at all

**Answer:** `C`

**Explanation:** When the criteria are objective and machine-checkable, a deterministic validator is the right evaluator: it is repeatable, variance-free, and immune to the self-bias that plagues LLM self-critique. It does not, however, judge semantic quality (that is where an LLM critic still helps), so the claim that it repairs meaning is wrong. Cost is not the core reason, and LLM critics certainly can run in loops — they are just weaker for objective constraints.

**Tags:** `Domain 1` `Advanced` `S6` `Workflow Patterns` `Structured Output`

### Q35. An S1 agent must enforce "no refund over $1,000 without human approval." The team encodes this only as a forceful system-prompt instruction. A jailbreak gets a $5,000 refund issued. What is the correct architecture?

- **A.** Make the prompt instruction even more forceful and add capital letters
- **B.** Ask the model to self-report whether it is about to exceed the limit
- **C.** Raise the iteration cap so the model has time to reconsider
- **D.** Enforce the limit in a deterministic pre-tool-use hook/guardrail that inspects the refund amount and blocks or routes to human approval regardless of model output; the prompt may restate intent but the binding guarantee lives in code

**Answer:** `D`

**Explanation:** A binding monetary policy is a critical business rule that must be enforced deterministically in code/hooks — relying on the prompt alone is anti-pattern #3 and is bypassable by jailbreaks, as the $5,000 refund shows. A stronger-worded prompt is still probabilistic, model self-reporting is anti-pattern #4 (uncalibrated and gameable), and the iteration cap is irrelevant to enforcement. Put the guarantee where the model cannot override it.

**Tags:** `Domain 1` `Advanced` `S1` `Deterministic Enforcement` `Escalation`
