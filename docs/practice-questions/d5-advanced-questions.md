# Domain D5 — Advanced & Edge-Case Questions

These are original, advanced and edge-case practice questions for **Domain 5 (Context Management & Reliability, 15%)** of the Claude Certified Architect – Foundations (CCA-F) exam. They are a personal study aid, not live or proctored exam content. They deliberately push past the base `d5-questions.md` bank to close coverage gaps: prompt-cache **minimum-token thresholds** and **write/read pricing multipliers**, **segment-level (not blanket) cache invalidation**, the **full `stop_reason` enumeration** (`refusal`, `stop_sequence`, `model_context_window_exceeded`, `pause_turn` for server tools, and empty `end_turn` responses), **Batch API operational edges** (result ordering via `custom_id`, 24-hour expiration, 29-day result retention, `max_tokens: 0` not supported, 100k/256 MB limits), **token-counting nuances** (estimate, free, does not use caching), and deeper compaction / context-isolation / per-slice-evaluation traps.

Each item states a single best answer and explains why the strongest distractor is wrong, naming the relevant anti-pattern where it applies. Difficulty and anchoring scenario (S1–S6) appear on the `Tags:` line.

Primary references used: platform.claude.com/docs (Prompt caching, Token counting, Batch processing, Messages API `stop_reason` / "Handling stop reasons"), Anthropic engineering "How we built our multi-agent research system" (anthropic.com/engineering/built-multi-agent-research-system), and Anthropic's context-engineering guidance.

## Prompt Caching: Minimum Token Thresholds and Silent No-Caching

### Q1. An architect marks a 600-token system prompt with a `cache_control` breakpoint on Claude Opus, expecting cache hits, but every request is billed at full input price. What is the most likely cause?

A) The prefix is below the model's minimum cacheable length, so the request is processed without caching and no error is returned
B) The breakpoint must be attached to a `messages` block, never to `system`
C) Opus does not support prompt caching at all
D) The cache silently expired between the two requests because the TTL is under one second
Answer: A
Explanation: Prompts shorter than the model's minimum cacheable length (on the order of ~1,024 tokens for some models and higher for others) are processed without caching, and the API returns no error — so a too-short prefix silently yields zero cache benefit. Caching works on `system` (and `tools`), Opus supports caching, and the default TTL is minutes, not sub-second.
Tags: D5 | Advanced | S1

### Q2. Why does Anthropic enforce a minimum number of tokens before a marked prefix is actually cached?

A) Because cache reads are billed at a premium, so short caches would lose money
B) Because caching very short prefixes provides little benefit relative to the bookkeeping cost; below the threshold the request is simply run without caching (no error)
C) Because short prompts cannot be tokenized
D) Because the API requires at least four breakpoints to enable caching
Answer: B
Explanation: There is a per-model minimum cacheable length; a prefix below it is processed normally with no caching and no error, because caching a tiny prefix is not worthwhile. Cache reads are *cheaper* (a fraction of base input price), short prompts tokenize fine, and one breakpoint is sufficient to request caching.
Tags: D5 | Advanced | S5

### Q3. A team splits a 900-token shared header into two cached segments with two breakpoints, hoping to cache each half. On their model the minimum cacheable length is larger than either half. What happens?

A) Each half is cached independently regardless of length
B) The API rejects the request with a 400 error for under-length cache blocks
C) Neither segment is large enough to meet the minimum cacheable length, so caching does not take effect for them, and the request runs without caching benefit
D) Only the second breakpoint is honored
Answer: C
Explanation: The minimum applies to the cached prefix length; splitting a small header into even smaller segments can push each below the threshold so nothing caches, silently. The fix is to cache the combined stable prefix as one segment over the threshold. No error is raised (it is silent), and the limit is about length, not breakpoint position.
Tags: D5 | Advanced | S5

## Prompt Caching: Write vs Read Pricing and Cost Modeling

### Q4. In Anthropic's prompt-caching pricing model, how do cache-write and cache-read token prices compare to the base input token price?

A) Cache writes cost more than base input (a multiplier above 1x), while cache reads cost a small fraction of base input
B) Both writes and reads are free
C) Cache writes are free and reads cost the full base input price
D) Reads cost more than writes, because reads are the common case
Answer: A
Explanation: Writing the cache costs a multiple of the base input price (more for the longer 1-hour TTL than the default 5-minute TTL), while reading a cache hit costs only a small fraction of base input — that asymmetry is exactly why caching pays off when a prefix is reused many times. Reads are cheaper than writes, and neither is free.
Tags: D5 | Advanced | S5

### Q5. An architect caches a large static prefix once but the workload only ever sends a single request before the cache expires. What is the cost outcome?

A) Caching always saves money on every request
B) The request is free because the first write is always free
C) The cache read price applies to the very first request
D) The single request paid the cache-write premium with no subsequent read to amortize it, so caching made that request more expensive, not cheaper
Answer: D
Explanation: Caching only pays off when the write premium is amortized over enough subsequent cheaper reads; a write with no reuse is strictly more expensive than not caching. The first call is a write (at a premium), not a read, and writes are not free — the break-even depends on how many reads follow.
Tags: D5 | Advanced | S5

### Q6. For a high-reuse stable document accessed many times over a long session with gaps exceeding five minutes, why might the 1-hour TTL be the cheaper choice despite its higher write price?

A) The 1-hour TTL makes reads free
B) Because re-writing the cache after each 5-minute expiry repeatedly pays the write premium, whereas one 1-hour write amortized over many reads avoids repeated writes
C) Because the 1-hour TTL disables billing for the session
D) Because the default TTL cannot be refreshed on a hit
Answer: B
Explanation: With gaps over five minutes, the default-TTL entry keeps expiring and forcing fresh (premium) writes; a single longer-lived write amortized over many cheap reads can be cheaper overall even though its write price is higher. Reads are never free, billing continues, and the default TTL *is* refreshed on each hit (the problem is the gap exceeds it).
Tags: D5 | Advanced | S5

## Prompt Caching: Segment-Level (Not Blanket) Invalidation

### Q7. A request caches the `tools` block under one breakpoint and the `system` block under a second. The architect edits only the system prompt. Which cache entries survive?

A) Both caches are invalidated because any edit nukes the whole request
B) Only the `system` cache survives; `tools` is invalidated
C) The `tools` cache survives; the `system` cache (and anything after it) is invalidated and re-encoded
D) Neither cache survives because system precedes tools in caching order
Answer: C
Explanation: Invalidation propagates forward from the changed block: editing `system` invalidates the system segment and everything after it, but the earlier `tools` segment (which is unchanged and sits before the edit) remains valid. This is exactly why multiple breakpoints exist — to isolate segments that change at different rates. Caching order is tools → system → messages, so an unchanged tools prefix survives a system edit.
Tags: D5 | Advanced | S5

### Q8. On a current-generation model (e.g., recent Opus/Sonnet) that preserves prior-turn thinking blocks, an architect runs a multi-turn agent with extended thinking enabled and a stable cached prefix. What is the most accurate statement about caching?

A) Enabling extended thinking permanently disables prompt caching
B) The cached stable prefix can keep getting hits across turns because the prior thinking blocks are preserved rather than stripped; changing the thinking configuration itself, however, invalidates the affected segments
C) Every turn invalidates the entire cache because thinking is involved
D) Thinking tokens are billed as cache writes
Answer: B
Explanation: On models that preserve prior-turn thinking blocks, the serialized prefix stays stable, so caching keeps working across turns; what invalidates a segment is *changing the thinking configuration* (a prefix change), which affects the system/messages segments rather than the unchanged tools segment. Thinking does not permanently disable caching, does not force a full invalidation every turn, and thinking tokens are not billed as cache writes.
Tags: D5 | Advanced | S5

### Q9. Which statement about how changing tool definitions interacts with the cache is correct?

A) Editing any tool definition invalidates the tools cache and every cached segment after it (system and messages), because tools sit first in the cached prefix
B) Editing a tool definition only invalidates the messages cache
C) Tool definitions are never part of the cached prefix
D) Tool edits invalidate only the specific tool's own block, leaving system and messages caches valid
Answer: A
Explanation: `tools` is the first component of the cached prefix (tools → system → messages), so any tool-definition change invalidates the tools segment and cascades forward to invalidate system and messages too. Tools are part of the cached prefix, and invalidation is forward-cascading from the earliest change, not isolated to a single tool block.
Tags: D5 | Advanced | S5

## stop_reason: The Full Enumeration and Correct Branching

### Q10. A long single-response generation returns `stop_reason: "model_context_window_exceeded"`. What does this mean and what is the correct handling?

A) The request failed with an HTTP 4xx error and must be retried
B) It means the same thing as `end_turn` — the task is complete
C) The model hit the context-window limit (not the `max_tokens` cap) before finishing; the returned content is valid but truncated, so handle it like a context-limit truncation rather than treating it as a complete answer
D) It signals a safety refusal
Answer: C
Explanation: `model_context_window_exceeded` is a *successful-response* stop reason (in the response body, not an HTTP error) indicating generation stopped at the context-window boundary rather than the `max_tokens` cap; the content is valid but incomplete, so you continue or surface the truncation. It is distinct from `end_turn` (natural completion) and from `refusal` (safety decline).
Tags: D5 | Advanced | S5

### Q11. How does `model_context_window_exceeded` differ from `max_tokens` as a stop reason?

A) They are identical and interchangeable
B) `max_tokens` means generation hit the per-request output cap you set; `model_context_window_exceeded` means generation hit the model's overall context-window limit even though `max_tokens` had room — both yield a valid but truncated response
C) `model_context_window_exceeded` is an error code while `max_tokens` is a stop reason
D) `max_tokens` always returns more output than `model_context_window_exceeded`
Answer: B
Explanation: `max_tokens` is your output cap being reached; `model_context_window_exceeded` is the model's total context budget being reached first — letting you request a large `max_tokens` without precomputing input size and still get a graceful truncation signal. Both are successful-response stop reasons, not errors, and which yields more output depends on the request.
Tags: D5 | Advanced | S5

### Q12. A response comes back with `stop_reason: "refusal"`. What is the correct architectural handling?

A) Treat it as a successful-response stop reason indicating the model declined on safety grounds; handle it distinctly (e.g., log, surface a safe message, or adjust the request) rather than retrying blindly or treating it as a normal completion
B) Parse the model's prose to decide whether it really refused
C) Retry the identical request immediately in a tight loop until it succeeds
D) Treat it as an HTTP 500 server error
Answer: A
Explanation: `refusal` is a distinct successful-response stop reason meaning the model declined for safety reasons; you branch on it explicitly rather than inferring refusal from text (anti-pattern 1) or mistaking it for an error or normal `end_turn`. Hammering the identical request in a tight loop is both ineffective and a backoff anti-pattern.
Tags: D5 | Advanced | S1

### Q13. An agent loop checks only for `tool_use` and `end_turn`, treating "anything else" as completion. Which real stop reason makes this loop silently ship a truncated or declined answer as if it were finished?

A) Only `tool_use` can ever appear, so the loop is safe
B) `max_tokens`, `model_context_window_exceeded`, `refusal`, and `pause_turn` are all "anything else" cases the loop would misread as completion, shipping truncated, declined, or paused responses as final
C) `end_turn` is the only stop reason, so nothing is missed
D) `stop_sequence` is the only other value and it always means completion
Answer: B
Explanation: A loop that buckets every non-`tool_use`, non-`end_turn` value as "done" mishandles `max_tokens` (truncated), `model_context_window_exceeded` (truncated), `refusal` (declined), and `pause_turn` (server-tool continuation needed) — each needs distinct handling. Treating them all as completion silently ships wrong or incomplete output, the reliability failure this domain targets.
Tags: D5 | Advanced | S3

### Q14. An agent using a server-side tool (e.g., web search) gets `stop_reason: "pause_turn"`. What is the correct action?

A) Treat the turn as complete and return the partial content to the user
B) Retry the original user request from scratch with backoff
C) Raise `max_tokens` and resend the original request
D) Continue the conversation by sending the assistant's response back so the model can finish the server-tool loop, since `pause_turn` means the server-side sampling loop reached its iteration limit mid-execution
Answer: D
Explanation: `pause_turn` indicates the server-side tool loop paused at its iteration limit; you append the assistant response to the conversation and make another request so the model can continue and finish. Treating it as complete drops in-progress work, and restarting from scratch or bumping `max_tokens` ignores the actual continuation mechanism.
Tags: D5 | Advanced | S4

### Q15. A `stop_sequence` stop reason is returned. What is the most accurate interpretation?

A) The model failed and the output should be discarded
B) It is identical to `end_turn` in all respects
C) The model emitted one of the caller's configured stop sequences and halted there; the response up to that point is valid, and you can inspect which sequence triggered it
D) The model ran out of context window
Answer: C
Explanation: `stop_sequence` means a caller-defined stop string was generated, so the model halted at that boundary; the content before it is valid and the triggering sequence is reported. It is not a failure, not the same as a natural `end_turn` (the cause differs), and unrelated to context-window exhaustion.
Tags: D5 | Intermediate | S6

### Q16. After sending tool results, an agent receives a response with `stop_reason: "end_turn"` but empty content. What is the recommended remediation?

A) Immediately resend the same empty response back to the model, expecting it to continue
B) Treat the empty response as a fatal error and crash the agent
C) Raise `max_tokens` so the empty content fills up
D) Avoid adding a text block immediately after the tool_result (which trains the model to end its turn); if it still occurs, add a continuation prompt in a new user message rather than re-sending the empty response
Answer: D
Explanation: Empty `end_turn` responses often follow adding text right after a `tool_result` (the model learns to expect user input there) or sending its finished response back unchanged; the fix is to not append text after tool results, and if needed prompt continuation in a *new* user message — re-sending the empty response does nothing because the model already decided it was done. It is not a crash condition, and `max_tokens` governs length, not emptiness.
Tags: D5 | Advanced | S1

### Q17. Why is `refusal` correctly classified as a `stop_reason` rather than an API error, and why does that distinction matter for reliability code?

A) It is actually an error; classifying it as a stop reason is wrong
B) It arrives in the response body of a successful (2xx) request, so your `try/except` around HTTP errors will not catch it — you must branch on `stop_reason` explicitly to handle it
C) It is a stop reason only when streaming
D) The distinction does not matter for code structure
Answer: B
Explanation: `refusal` is part of a successful response body, so error-handling blocks built for HTTP 4xx/5xx will not see it; you must inspect `stop_reason` in your normal response path. Conflating stop reasons with errors leaves refusals (and truncations) unhandled — a reliability gap regardless of streaming.
Tags: D5 | Advanced | S1

## Batch API: Result Ordering, custom_id, Expiration, and Limits

### Q18. An S5 batch returns and the architect zips results to inputs by position (first result = first request). Reviews start getting attached to the wrong files. What is the root cause and fix?

A) Batch results are not guaranteed to come back in submission order; match each result to its request by the `custom_id` you assigned, not by position
B) Batches always preserve order; the bug must be elsewhere
C) The fix is to submit one request per batch so order is trivially preserved
D) Sort results alphabetically by content to recover order
Answer: A
Explanation: Batch results can return in any order, so positional zipping silently mismatches results to inputs; you must correlate via the `custom_id` set on each request. Submitting one-per-batch defeats the point of batching, and sorting by content is a fragile workaround, not the documented correlation mechanism.
Tags: D5 | Advanced | S5

### Q19. A batch is submitted but a subset of requests is still unprocessed at the 24-hour mark. What happens to those requests?

A) They keep processing indefinitely until they finish
B) The entire batch, including completed results, is deleted at 24 hours
C) The batch expires at 24 hours; requests not completed by then end as `expired` and are not billed, while completed results remain retrievable
D) Expired requests are automatically retried in a new batch at no cost
Answer: C
Explanation: A batch expires if processing does not complete within 24 hours; unfinished requests are marked `expired` (and are not billed), while already-completed results are still available. Completed results are not deleted at expiration, and there is no automatic re-batching — you resubmit the expired items yourself.
Tags: D5 | Advanced | S5

### Q20. How long are completed Batch API results available for download, and what is the operational implication?

A) Indefinitely; you can fetch them any time
B) For exactly 24 hours, the same as the processing window
C) Only while the batch `processing_status` is `in_progress`
D) For about 29 days after batch creation; after that you can still view the batch but can no longer download results, so persist results to your own storage promptly
Answer: D
Explanation: Results are retrievable for roughly 29 days after creation; beyond that the batch is viewable but results are no longer downloadable, so a reliable pipeline persists results to durable storage soon after completion. The retention window is not indefinite, is longer than the 24-hour processing window, and results are fetched after status becomes `ended`, not during `in_progress`.
Tags: D5 | Advanced | S5

### Q21. What are the documented size limits of a single Message Batch?

A) Up to 100,000 requests or 256 MB in size, whichever limit is reached first
B) Unlimited requests up to 1 GB
C) Exactly 1,000 requests, no size limit
D) Up to 10,000 requests or 1 MB
Answer: A
Explanation: A batch is capped at 100,000 requests or 256 MB, whichever comes first, so very large workloads must be chunked across multiple batches. The other figures are fabricated.
Tags: D5 | Advanced | S5

### Q22. An architect wants to pre-warm a prompt cache by issuing a `max_tokens: 0` request inside a batch. Why does this fail?

A) `max_tokens: 0` is the normal way to pre-warm inside batches and works fine
B) Batches do not support caching at all
C) Each batched request must have `max_tokens` of at least 1; `max_tokens: 0` cache pre-warming is not supported in a batch because an ephemeral cache entry written during batch processing would likely expire before the follow-up request runs
D) Pre-warming requires a 1-hour TTL, which batches forbid
Answer: C
Explanation: Batched requests require `max_tokens >= 1`, and `max_tokens: 0` pre-warming is explicitly unsupported in batches because the ephemeral cache would expire before the dependent request executes asynchronously. Batches can still benefit from caching within a request; the unsupported case is specifically the `max_tokens: 0` pre-warm pattern.
Tags: D5 | Advanced | S5

### Q23. How should code monitor a batch for completion before retrieving results?

A) Wait a fixed 24 hours, then fetch, since that is the max processing time
B) Open a websocket and block until a push notification arrives
C) Re-create the batch repeatedly until results appear
D) Poll the batch's `processing_status` until it becomes `ended`, then retrieve results (streaming them if large), matching each by `custom_id`
Answer: D
Explanation: The correct pattern is to poll `processing_status` until `ended` (most batches finish well under the 24-hour ceiling), then stream/download results and correlate via `custom_id`. Blindly waiting the full 24 hours wastes time, there is no blocking websocket for this, and re-creating the batch duplicates work.
Tags: D5 | Intermediate | S5

### Q24. A batch is canceled mid-processing. What is the correct expectation about its results?

A) Cancellation finalizes to an `ended` status and the batch may contain partial results for requests already processed before cancellation; correlate those by `custom_id`
B) All results are discarded and the batch is unusable
C) Cancellation is impossible once a batch is submitted
D) Canceled requests are billed at double rate as a penalty
Answer: A
Explanation: After cancellation the batch settles to `ended` and can hold partial results for requests completed before the cancel took effect, which you retrieve and match by `custom_id`. Cancellation is supported, results are not all discarded, and there is no penalty billing for canceled (unsent) requests.
Tags: D5 | Advanced | S5

## Token Counting: Estimate, Free, No Caching

### Q25. An architect calls the token-counting endpoint with `cache_control` blocks included, expecting it to warm the cache. What actually happens?

A) The token-counting call writes the cache, so the next message-create is a cache hit
B) Including `cache_control` makes token counting return an error
C) Token counting returns a token estimate and does not perform prompt caching; caching only occurs during actual message creation, even if `cache_control` is present in the count request
D) Token counting always uses the cache, so counts reflect cached pricing
Answer: C
Explanation: The token-counting endpoint estimates input tokens and does not run caching logic; you may include `cache_control` blocks but no cache is written — caching happens only on real message creation. It does not error on `cache_control`, and it does not pre-warm the cache.
Tags: D5 | Advanced | S6

### Q26. The token-counting endpoint returns 14,000 input tokens, but the actual message-create request bills a slightly different number. Why, and what is the takeaway?

A) The endpoint is buggy and the count should be ignored
B) The discrepancy means caching silently activated during counting
C) Counts are always exact, so any difference indicates a request mismatch
D) The count is an estimate that can differ from the billed amount by a small margin (and may include system-added tokens you are not billed for), so reserve output headroom and a small safety margin rather than treating the count as exact
Answer: D
Explanation: Token counts are explicitly estimates and can differ slightly from actual usage (and may include system-added tokens you are not billed for), so budget with a margin and reserve output headroom. The endpoint is not buggy, counting does not activate caching, and the small expected variance is normal, not a sign of a mismatched request.
Tags: D5 | Advanced | S6

### Q27. Which best describes the cost and capability profile of the token-counting endpoint for budgeting a request?

A) It is billed per call at the input rate, so counting is as expensive as sending
B) It is free to use (subject to its own rate limits) and accepts the same structured inputs as message creation — system, tools, images, and PDFs — so you can measure the full prompt before sending
C) It only counts the plain user text and ignores tools and system
D) It must be called once per token to tally the total
Answer: B
Explanation: Token counting is free (with its own RPM limits independent of message-creation limits) and accepts the same inputs — system, tools, images, PDFs — so you can size the *entire* prompt, including the fast-growing components. It is not billed at the input rate, does count tools and system, and returns a single total, not one call per token.
Tags: D5 | Advanced | S5

## Compaction, Isolation, and Per-Slice Evaluation (Deeper Edges)

### Q28. A coordinator compacts at exactly 100% of the window, but compaction itself needs tokens to generate the summary and the request overflows before the summary is produced. What is the design fix?

A) Trigger compaction at a threshold below the window limit (reserving headroom for the summary generation and continued output), not at the moment of overflow
B) Compact only after the window is completely full, to maximize history retained
C) Disable `max_tokens` so the summary can be any length
D) Switch to a larger model whenever compaction is needed
Answer: A
Explanation: Compaction consumes budget to produce the summary plus headroom to continue, so the threshold must fire *before* the window is exhausted; compacting at 100% leaves no room to even generate the summary. Removing `max_tokens` does not create input room, and a bigger model just delays the same overflow without fixing the trigger logic.
Tags: D5 | Advanced | S3

### Q29. After compaction, a subagent re-reads the running findings file and finds a stale entry that contradicts a newer tool result. What is the most robust pattern to prevent acting on stale external state?

A) Trust the conversation history over the file, since history is freshest
B) Stop using external memory because it can go stale
C) Treat the external store as the source of truth but version/timestamp entries and reconcile on read (last-writer-wins or explicit supersession), so the agent acts on current state rather than a stale note
D) Re-run the entire task from scratch whenever any conflict appears
Answer: C
Explanation: External memory is durable but must be kept coherent: version or timestamp entries and reconcile on read so the agent uses current state, not a stale note. Trusting volatile history defeats the durability of external memory; abandoning external memory loses the durability benefit; and full restarts on every conflict waste budget instead of reconciling.
Tags: D5 | Advanced | S3

### Q30. A coordinator spawns 8 subagents and, to "be safe," also keeps each subagent's full transcript in a side log it re-injects into its own next prompt. Performance degrades. What principle is violated and what is the fix?

A) Nothing is violated; more context is always safer
B) The fix is to spawn fewer subagents so there is less to re-inject
C) The fix is to stream the transcripts faster so they fit
D) Context isolation is violated — re-injecting full subagent transcripts reintroduces the noise isolation removed; keep transcripts in durable external storage for audit, but feed only the condensed findings back into the coordinator's window
Answer: D
Explanation: Re-injecting full subagent transcripts into the coordinator's window defeats context isolation and causes the rot it was meant to prevent; audit logs belong in external storage, while only condensed findings cross back into the active window. Fewer subagents and faster streaming both miss the point — the problem is *what* enters the window, not how many agents or how fast.
Tags: D5 | Advanced | S3

### Q31. An S6 extraction eval reports 96% aggregate accuracy, and per-document-type breakdown shows invoices at 99% but a rare "amended contract" type at 41% on a 12-sample slice. The team says "12 samples is too few to matter." What is the correct stance?

A) The low-n slice is a signal of a likely failing high-stakes category, not noise to dismiss; gather more samples for that slice and gate on the worst slice rather than letting the aggregate mask it
B) Agree — small slices should be excluded from gating
C) Re-weight the rare type out of the metric so the aggregate looks clean
D) Ship now and monitor the rare type in production only
Answer: A
Explanation: A worst-slice at 41% is exactly what aggregate accuracy hides (anti-pattern 10); a small, high-stakes slice warrants *more* evaluation, not exclusion, and gating on the worst slice catches the failure pre-ship. Excluding or re-weighting the slice deliberately hides the failure, and shipping-then-watching ships a known-bad category.
Tags: D5 | Advanced | S6

### Q32. Why can a tool that returns `[]` on error specifically corrupt per-document-type accuracy metrics (not just hide a single failure)?

A) It cannot affect metrics; empty results are neutral
B) Empty results are always counted as failures, deflating accuracy
C) An error-as-empty-result is silently counted as a successful "nothing found" outcome, so the failing documents are scored as passes — inflating the slice's apparent accuracy and erasing the very failures the per-slice metric exists to surface
D) Metrics ignore empty results entirely, so there is no effect
Answer: C
Explanation: Suppressing errors as empty results (anti-pattern 7) makes failures indistinguishable from legitimate empty outcomes, so they score as passes and inflate the slice's accuracy — per-slice metrics only work if failures are recorded, not swallowed. The effect is inflation (failures counted as passes), the opposite of being counted as failures or ignored.
Tags: D5 | Advanced | S5

### Q33. A multi-pass CI review (S5) runs pass 2 in the same session as pass 1 to "save tokens by reusing context." Reviews miss bugs the author introduced. What is the context-management diagnosis and remedy?

A) Pass 2 needs a higher `max_tokens`
B) The two passes should share even more context for consistency
C) Replace pass 2 with the author rating its own confidence
D) Same-session self-review (anti-pattern 9) lets the reviewer inherit the author's reasoning bias; run the review pass in a fresh, isolated context so it evaluates the artifact independently, and report findings per check type
Answer: D
Explanation: Reusing the author's session inherits its bias, so the reviewer rationalizes the same mistakes; a fresh isolated context yields an independent review, and per-check-type reporting surfaces category misses. A bigger `max_tokens` does not remove bias, more shared context worsens it, and self-reported confidence (anti-pattern 4) is exactly the soft signal to avoid.
Tags: D5 | Advanced | S5

### Q34. A pipeline retries a failed extraction with backoff, but the retried "write validated record" step double-writes because the first attempt actually succeeded before the timeout. What two patterns together fix this correctly?

A) Make the write idempotent (idempotency key or natural key) so a retry cannot double-write, and surface the original failure context so the retry decision is informed rather than blind
B) Increase the timeout so attempts never overlap, and trust prose confirmation from the model
C) Lower the retry count to zero so no retries ever happen
D) Suppress the duplicate silently downstream with a dedupe filter as the primary fix
Answer: A
Explanation: Idempotency makes the retried write safe regardless of whether the first landed, and surfacing the real failure context informs whether/how to retry — together they fix the root cause. A longer timeout never guarantees the outcome is known, removing retries sacrifices resilience, and a downstream dedupe filter is a workaround masking the missing idempotency rather than fixing it.
Tags: D5 | Advanced | S6

### Q35. An agent's tool returns a 12,000-token result every call; the architect "fixes" rot by raising the compaction frequency so the window is summarized after every tool call. Throughput collapses. What is the better root-cause fix?

A) Compact even more aggressively, after every model token
B) Raise the context window and keep the 12,000-token results
C) Make the tool return a lean structured payload (only needed fields/IDs) and store the full result externally behind a handle the agent can re-fetch on demand, so the window never ingests the bloat in the first place
D) Tell the model in the system prompt to ignore most of the tool output
Answer: C
Explanation: Constant compaction treats the symptom (bloated window) while paying a summarization tax every call; the root cause is the tool dumping low-signal tokens, fixed by lean payloads plus externalized large results re-fetched by reference. A bigger window only delays rot, and prompting the model to ignore fields is non-deterministic and still pays the token cost.
Tags: D5 | Advanced | S4

### Q36. A graceful-degradation path returns partial results when a dependency is down, but labels them as complete to avoid alarming users. A downstream agent then acts on them as if whole. What rule was broken?

A) None — hiding partiality keeps the UX smooth
B) The system should have crashed instead of degrading
C) Partial results should always be discarded
D) Degraded results must be clearly labeled as partial/degraded so downstream consumers (and the model) can branch correctly; mislabeling partial results as complete is a form of suppressing the failure, which causes silent wrong decisions
Answer: D
Explanation: Graceful degradation requires honest labeling so downstream logic treats partial output as partial; relabeling it complete is a suppression anti-pattern that propagates silent errors. Crashing is not the only alternative, and discarding usable partial results forfeits the value degradation was meant to preserve — the fix is correct labeling, not hiding or dumping.
Tags: D5 | Advanced | S1

### Q37. Which design correctly combines `stop_reason` branching with state verification to declare an agentic task complete?

A) Stop as soon as the model writes "done" anywhere in its output
B) Stop when `stop_reason` is `end_turn` AND a deterministic check confirms required state (e.g., expected files written, validated records persisted) — and explicitly handle `tool_use`, `max_tokens`, `model_context_window_exceeded`, `refusal`, and `pause_turn` rather than treating them as completion
C) Stop only when the iteration cap is hit
D) Stop when the model's self-reported confidence exceeds 0.9
Answer: B
Explanation: Robust completion pairs `stop_reason == end_turn` with a deterministic state check, while every other stop reason gets its own branch (tool execution, truncation handling, refusal handling, server-tool continuation). Parsing prose (anti-pattern 1), relying on the cap (anti-pattern 2), and self-reported confidence (anti-pattern 4) are all the soft signals this domain rejects.
Tags: D5 | Advanced | S3

### Q38. An architect caps retrieved context per turn at a fixed token budget, but occasionally the single most relevant chunk exceeds the cap and gets dropped, causing wrong answers. What is the better budgeting approach?

A) Remove the per-turn cap entirely and send everything retrieved
B) Always truncate the longest chunk to fit, regardless of relevance
C) Rank retrieved chunks by relevance and fit the highest-signal ones within budget (summarizing or externalizing overflow with a re-fetch handle), so the cap protects the window without silently dropping the load-bearing chunk
D) Raise `max_tokens` so the dropped chunk reappears
Answer: C
Explanation: A budget should be filled by relevance, not arbitrary order or length, so the most load-bearing chunk is never the one silently dropped; overflow can be summarized or externalized behind a handle. Removing the cap reintroduces rot, truncating by length ignores relevance, and `max_tokens` controls output, not which input chunks are retained.
Tags: D5 | Advanced | S4

### Q39. In an S3 system, a subagent legitimately finds nothing relevant. How should this be represented to the coordinator so reliability and metrics are preserved?

A) Return an empty result identical to a tool error, so the coordinator cannot tell them apart
B) Return nothing at all and let the coordinator assume the worst
C) Fabricate a marginally relevant result to avoid an empty hand-off
D) Return an explicit, structured "no relevant results found, here is what was searched and why" — distinct from an error state — so the coordinator can reason correctly and per-slice metrics can separate true-negatives from failures
Answer: D
Explanation: A genuine negative and an error are different states; representing the negative explicitly (with what was searched) lets the coordinator branch correctly and lets metrics distinguish true negatives from suppressed failures. Making a negative indistinguishable from an error blinds the coordinator, returning nothing suppresses information, and fabricating a result corrupts synthesis.
Tags: D5 | Advanced | S3

### Q40. An architect wants the cheapest reliable design for an S5 nightly review of 40,000 files with no latency requirement. Which combination is correct?

A) Use the Batch API (chunking to respect the 100,000-request / 256 MB limits), correlate results by `custom_id`, retry only `expired`/transient items with backoff and idempotency, persist results before the 29-day window lapses, and report accuracy per check type and file class
B) Fire 40,000 synchronous Messages calls in a tight loop with no backoff
C) Concatenate all 40,000 files into one Messages request to amortize overhead
D) Stream all 40,000 responses simultaneously to finish fastest
Answer: A
Explanation: A latency-tolerant bulk job is the Batch API's sweet spot (50% cheaper), with `custom_id` correlation, targeted retries of expired/transient items, durable result persistence within retention, and per-slice metrics — every D5 reliability lever at once. A synchronous tight loop invites rate limits, one giant concatenated request floods the window (context rot), and mass simultaneous streaming overwhelms limits and complicates reconciliation.
Tags: D5 | Advanced | S5

## Answer Key (quick reference)

Q1 A · Q2 B · Q3 C · Q4 A · Q5 D · Q6 B · Q7 C · Q8 B · Q9 A · Q10 C · Q11 B · Q12 A · Q13 B · Q14 D · Q15 C · Q16 D · Q17 B · Q18 A · Q19 C · Q20 D · Q21 A · Q22 C · Q23 D · Q24 A · Q25 C · Q26 D · Q27 B · Q28 A · Q29 C · Q30 D · Q31 A · Q32 C · Q33 D · Q34 A · Q35 C · Q36 D · Q37 B · Q38 C · Q39 D · Q40 A
</content>
</invoke>
