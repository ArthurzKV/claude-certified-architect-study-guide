# Domain 5 — Context Management & Reliability: Practice Questions

These are original, scenario-based practice questions for **Domain 5 (Context Management & Reliability, 15%)** of the Claude Certified Architect – Foundations (CCA-F) exam. They are a personal study aid, not live or proctored exam content. Each question states a single best answer and explains why the strongest distractor is wrong, naming the relevant anti-pattern. Difficulty and anchoring scenario (S1–S6) appear in the `Tags:` line. Topics covered: the CALM context lifecycle, context rot, prompt caching (`cache_control` breakpoints, ordering, TTLs, invalidation), conversation compaction, structured note-taking / external memory, token budgeting, sub-agent context isolation, and reliability (backoff, idempotency, surfacing errors, per-category metrics).

Primary references used: platform.claude.com/docs (Prompt caching, Token counting, Message Batches, Messages API `stop_reason`), Anthropic engineering "How we built our multi-agent research system" (anthropic.com/engineering/built-multi-agent-research-system), and Anthropic's context-engineering guidance.

## Context Lifecycle and Context Rot (CALM)

### Q1. A long-running S3 coordinator agent "starts ignoring earlier instructions" after roughly 40 turns of accumulated tool output. What is the architecturally correct first intervention?
A) Swap to a model with a larger context window and resend the full transcript every turn
B) Add a system-prompt sentence pleading with the model to "remember all earlier instructions"
C) Raise the per-task iteration cap so the agent has more turns to recover
D) Apply context engineering — compact stale history, isolate subagent work, and externalize state — to reduce low-signal tokens in the window
Answer: D
Explanation: Degradation after many turns is context rot, fixed by context engineering (compaction, isolation, external memory), not by more turns or a bigger window. Raising the iteration cap (anti-pattern 2) treats a control-flow knob as a reliability fix; a bigger window still fills with the same low-signal tokens; prompt pleading (anti-pattern 3) is not deterministic enforcement.
Tags: D5 | Intermediate | S3

### Q2. What does the CALM mnemonic (Curate, Allocate, Limit, Maintain) primarily encourage an architect to treat the context window as?
A) A cache that the API manages automatically with no architect involvement
B) Free space that should be filled to maximize the model's available information
C) An immutable log that must contain the complete conversation for auditability
D) A finite, deliberately managed resource where you decide which tokens occupy the window at each step
Answer: D
Explanation: CALM frames context engineering: curate what enters, allocate a token budget, limit what persists turn-to-turn, and maintain state externally. The window is a budget, not free space — the "fill it up to maximize information" choice is the trap that causes context rot; auditability is served by external logs, and caching is opt-in, not automatic.
Tags: D5 | Foundational | S3

### Q3. A scenario claims "Claude supports a long context window, so include the entire 200-page operations manual on every turn." Why is this the wrong design?
A) The manual must be uploaded via the Files API or it cannot be referenced
B) The API rejects any request over 100 pages of input
C) Flooding the window with low-signal tokens causes context rot and raises cost and latency; retrieve or summarize only the relevant sections instead
D) Long documents cannot be tokenized reliably
Answer: C
Explanation: A large window is a budget, not free space; dumping the whole manual every turn degrades retrieval/instruction-following and inflates cost and time-to-first-token. There is no hard "100 page" rule, tokenization works fine, and Files API is one delivery mechanism, not the reason the design is wrong.
Tags: D5 | Foundational | S4

### Q4. In the documented context lifecycle, which ordering of components (most stable to most volatile) lets an architect reason about what to cache versus compact?
A) Few-shot examples → tools schema → running conversation → system prompt
B) System prompt → running conversation → tools schema → few-shot examples
C) Tools schema → system prompt → few-shot examples → retrieved/long context → running conversation
D) Running conversation → retrieved context → few-shot examples → system prompt → tools schema
Answer: C
Explanation: The lifecycle runs stable-to-volatile: tools schema, then system prompt, then few-shot examples, then retrieved/long context, then the running conversation. This ordering is exactly what makes the stable prefix cacheable and the volatile tail compactable. Reversed orders mix volatile content into the prefix and break caching.
Tags: D5 | Intermediate | S3

### Q5. Why is "right altitude" the goal when assembling context for an agent step?
A) Because the smallest set of high-signal tokens needed to decide correctly minimizes attention errors, latency, and cost
B) Because shorter prompts always produce shorter, cheaper outputs
C) Because the model performs best with the absolute maximum number of tokens
D) Because the API charges a flat rate regardless of input size
Answer: A
Explanation: Right altitude means enough context to decide, not a data dump; every added token costs attention, latency, and dollars and adds a chance to attend to the wrong thing. Output length is not strictly a function of input length, and input is billed per token, so a flat-rate assumption is false.
Tags: D5 | Foundational | S4

## Prompt Caching: cache_control Breakpoints, Ordering, TTLs, Invalidation

### Q6. A multi-turn agent has a large fixed system prompt and a stable tool set, but the latest user turn changes every call. Where should the single `cache_control` breakpoint go?
A) Caching is not possible when any part of the request changes between calls
B) After the stable tools-plus-system prefix, with the dynamic user turn placed below the breakpoint
C) On the latest user message, so the most recent content is cached
D) On the first few-shot example only, splitting it from the system prompt
Answer: B
Explanation: Cache the stable prefix and keep volatile content below the last breakpoint; the prefix is matched token-for-token, so a breakpoint after tools+system gives reusable hits while the changing user turn lives underneath. Putting the breakpoint on the latest user message caches nothing reusable (the classic mis-placement), and partial-change requests are exactly what caching is for.
Tags: D5 | Intermediate | S1

### Q7. What is the maximum number of `cache_control` breakpoints you can place in a single request, and why use more than one?
A) Unlimited breakpoints; the API caches every block automatically
B) Up to 4 breakpoints; use several to cache segments that change at different rates so editing a later segment does not invalidate an earlier one
C) Up to 10 breakpoints; one per message is required
D) 1 breakpoint; multiples are not supported
Answer: B
Explanation: You can place up to 4 breakpoints; multiple breakpoints let you cache a stable tools prefix separately from, say, a large document, so changing the document does not invalidate the tools cache. Caching is opt-in (not automatic per block), and there is no per-message requirement.
Tags: D5 | Intermediate | S5

### Q8. With prompt caching, what exactly becomes the cached prefix when you attach an ephemeral `cache_control` breakpoint to a block?
A) Everything after the breakpoint, since that is what changes least
B) The entire request including the model's eventual output
C) Only the single block the breakpoint is attached to
D) Everything before and including that breakpoint, matched as an exact token-for-token prefix
Answer: D
Explanation: The breakpoint marks the end of the cacheable prefix: everything before and including it is cached and matched exactly token-for-token. A change anywhere in that prefix invalidates the cache. Caching only the single block, or the suffix, misreads the prefix-matching model; outputs are never part of the cached input.
Tags: D5 | Intermediate | S1

### Q9. An S5 CI/CD pipeline reuses a large static coding-standards document and tool set across many file reviews, but reviews of a given file are spaced more than five minutes apart during a long run. Which cache TTL choice is most appropriate?
A) Re-upload the document each call to force a fresh cache write
B) Disable caching, because gaps over five minutes make caching impossible
C) The default 5-minute TTL, because it is always cheapest
D) The longer 1-hour TTL, because the stable content is reused over a long window with gaps exceeding the default lifetime
Answer: D
Explanation: Choose the 1-hour TTL when large stable content is hit repeatedly across a session but with gaps longer than the 5-minute default; the default suits tight loops. Caching still works across gaps (the entry just must not have expired), and re-uploading defeats the purpose by forcing repeated cache writes.
Tags: D5 | Intermediate | S5

### Q10. Which change will invalidate a cached prefix and force re-encoding of everything from that point onward?
A) Lowering `max_tokens` for the model's output
B) Reading a cache hit (each hit refreshes the entry)
C) Appending a new user message strictly below the final cache breakpoint
D) Editing a tool definition that sits at or before the breakpoint
Answer: D
Explanation: Any edit to content at or before a breakpoint — a tool definition, the system prompt, reordering blocks, or toggling settings that alter the serialized prefix — invalidates that cache and everything after it. Appending below the last breakpoint is exactly the safe pattern; cache hits refresh rather than invalidate; and `max_tokens` controls output, not the cached input prefix.
Tags: D5 | Advanced | S5

### Q11. Toggling extended thinking on or off between two otherwise identical requests unexpectedly causes cache misses. What is the most accurate explanation?
A) Extended thinking disables prompt caching entirely and permanently
B) The cache only works for requests with no thinking blocks
C) Thinking tokens are billed as cache writes, doubling the cost
D) Enabling/disabling thinking changes the serialized prefix, so it invalidates the previously cached prefix like any other prefix edit
Answer: D
Explanation: Caching matches an exact serialized prefix; toggling extended thinking (or changing tool definitions) alters that prefix and invalidates the cache, just like editing the system prompt. It does not permanently disable caching, thinking tokens are not billed as cache writes, and caching is compatible with thinking when the prefix is held stable.
Tags: D5 | Advanced | S5

### Q12. Where should volatile per-turn data (the latest user message, freshly retrieved snippets) be placed relative to the final `cache_control` breakpoint?
A) It does not matter; the cache ignores message position
B) Above the breakpoint, interleaved with the system prompt for relevance
C) Below the final breakpoint, so it never alters the cached prefix
D) Inside the tool definitions, so tools always see the latest data
Answer: C
Explanation: Volatile content must go below the final breakpoint so it never breaks the cached prefix; interleaving dynamic data into the prefix causes constant invalidation (a named caching anti-pattern). Position absolutely matters for prefix matching, and embedding turn data into tool definitions would invalidate the tool cache every turn.
Tags: D5 | Intermediate | S1

### Q13. An agent's per-turn retrieved snippets change every call. To preserve cache hits on the stable tools+system+examples header, what is the correct structure?
A) Put retrieved snippets at the top, then tools, then system, then the breakpoint
B) Disable caching whenever retrieval is involved
C) Add a separate breakpoint after every retrieved snippet so each one is cached
D) Keep tools → system → few-shot as a cached prefix ending in a breakpoint, then append the changing retrieved snippets and user turn below it
Answer: D
Explanation: Order most-stable-to-least-stable and cache the stable header; volatile retrieval lives below the last breakpoint. Putting changing snippets at the top destroys the prefix; caching per-snippet wastes breakpoints on content that changes every call; and retrieval does not preclude caching the stable portion.
Tags: D5 | Advanced | S4

## Conversation Compaction and Note-Taking / Memory

### Q14. The S3 coordinator is about to exhaust its context window mid-research. What is the correct compaction strategy?
A) Truncate the oldest messages blindly until the window fits
B) Restart the research task from scratch with an empty context
C) Generate a structured, state-preserving summary (findings so far, sources cited, sub-questions remaining) and continue from that summary plus the most recent turns
D) Switch every subagent to a larger model and resend all transcripts
Answer: C
Explanation: Compaction should preserve key state — decisions, open tasks, constraints, IDs — then continue. Blind truncation drops the task definition (the oldest, most load-bearing message); restarting loses all progress; and a bigger model does not address an unbounded, low-signal history.
Tags: D5 | Intermediate | S3

### Q15. What distinguishes a good compaction summary from a bad ("lossy") one?
A) The good one is state-preserving — it keeps the identifiers, decisions, and commitments the agent needs to act, dropping only conversational filler
B) The good one is prose-pretty and reads like a polished paragraph
C) The good one preserves the verbatim text of every prior tool output
D) The good one is the shortest possible, even if it omits file paths and IDs
Answer: A
Explanation: A good summary keeps the actionable state (decisions made, open tasks, constraints, tool results that still matter, file paths/IDs) and drops only filler. Prose polish is irrelevant; aggressively shortening past the IDs is the lossy trap; and preserving every verbatim tool output defeats the purpose of compaction.
Tags: D5 | Intermediate | S3

### Q16. In Claude Code, which built-in behavior summarizes a long session so work can continue, and how is the same effect achieved in the Messages API?
A) `/compact` in Claude Code; in the Messages API you must rely on the API to auto-summarize with no code
B) `/clear` in Claude Code wipes and resummarizes; the Messages API has an identical `/clear` endpoint
C) There is no compaction in Claude Code; you always start a new session
D) `/compact` in Claude Code; in the Messages API (and Agent SDK) you implement state-preserving summarization yourself, or use built-in context/memory features where available
Answer: D
Explanation: `/compact` performs summarize-and-continue in Claude Code; in the Messages API and Agent SDK you implement compaction (or use available context/memory features). The Messages API does not auto-summarize for you, `/clear` discards rather than compacts, and Claude Code does support compaction.
Tags: D5 | Foundational | S2

### Q17. What is the most robust pattern for a multi-hour agent to avoid losing progress when its context is compacted?
A) Run every step twice and compare the two transcripts
B) Keep all state inside the conversation history and hope compaction preserves it
C) Increase `max_tokens` so output never gets cut off
D) Persist state to external memory (a notes file, scratchpad, task list, or memory tool) and reload only what the current step needs
Answer: D
Explanation: External memory keeps the source of truth outside the window so state survives compaction, retries, and crashes; the agent reads back only what it needs. Relying on conversation history is exactly what compaction can lose; `max_tokens` governs output length, not durable state; and double-running wastes budget without persisting anything.
Tags: D5 | Intermediate | S3

### Q18. In an S6 extraction pipeline that processes thousands of records, how should validated records be handled to keep the working set bounded?
A) Re-send all prior records on every turn so the model maintains continuity
B) Keep all records in the system prompt so the model can cross-check them
C) Hold every validated record in the conversation until the whole batch finishes, then return them all at once
D) Write each validated record out to external storage immediately and keep only a small working set plus references in context
Answer: D
Explanation: External memory turns an unbounded conversation into a bounded working set plus durable storage; writing each validated record out immediately keeps the window small. Holding all records in context (or in the system prompt, or re-sending them) accumulates low-signal tokens and causes context rot and cost blowup.
Tags: D5 | Intermediate | S6

### Q19. Which best describes structured note-taking as a long-horizon reliability pattern?
A) The agent appends every thought to the conversation so nothing is ever lost
B) The agent stores notes in `max_tokens` reserved output space
C) The agent keeps notes only in its own reasoning, never written down
D) The agent writes structured notes (file, scratchpad, memory tool, or task list) and reads back only what the current step requires, keeping the source of truth outside the window
Answer: D
Explanation: Structured note-taking externalizes the source of truth and reads back selectively, making state durable across compaction and crashes. Notes kept only in reasoning vanish on compaction; appending every thought to the conversation accelerates context rot; and `max_tokens` is an output cap, not a storage location.
Tags: D5 | Foundational | S3

## Token Budgeting and Estimation

### Q20. To gate whether a request fits the window, a teammate hard-codes "1 token ≈ 4 characters." Why is the measured approach better?
A) The model rejects any request the architect did not pre-tokenize
B) Character ratios are always exactly 4, so measuring is redundant
C) Token-per-character ratios vary by content (code, JSON, non-English), so use the token-counting tool/endpoint and reserve output headroom to avoid silent overflow
D) Tokens are billed by character, so counting tokens is unnecessary
Answer: C
Explanation: Anthropic's guidance is to count tokens, not guess; ratios differ across code, JSON, and non-English text, so a hard-coded ratio silently overflows. Use the token-counting utility and reserve headroom for output and tool-result growth. Ratios are not constant, the API does not require client-side pre-tokenization, and billing is per token, not per character.
Tags: D5 | Intermediate | S6

### Q21. When budgeting tokens for a turn, which "directions" of consumption must be accounted for besides the static prompt?
A) Only the user's typed message length
B) Only the number of tools defined, regardless of their output size
C) Only the system prompt; everything else is negligible
D) Output (`max_tokens`), extended-thinking tokens, and tool-result growth — tool outputs are often the fastest-growing, lowest-signal part of the window
Answer: D
Explanation: Budget both directions: reserve output headroom via `max_tokens`, account for thinking tokens, and especially for tool-result growth, which fills windows fastest. The system prompt alone, the user message alone, or the tool count alone all ignore the dominant, fast-growing tool-output consumption.
Tags: D5 | Intermediate | S5

### Q22. Which is a sound practical token budget for a long-running agent?
A) Cap and cache the static prefix, cap retrieved context per turn, and set a compaction threshold (e.g., compact when used context crosses a chosen fraction of the window)
B) Disable tool use to keep the window small
C) Set `max_tokens` to the model's maximum so output is never limited
D) Send the full window every turn and let the API truncate as needed
Answer: A
Explanation: A disciplined budget caps and caches the stable prefix, caps per-turn retrieval, and compacts at a threshold fraction of the window. Letting the API truncate drops load-bearing content unpredictably; maxing `max_tokens` starves input budget; and disabling tools removes capability rather than managing budget.
Tags: D5 | Intermediate | S3

### Q23. Why are lean, structured tool results (IDs and needed fields) preferred over raw data dumps?
A) Raw dumps are rejected by the Messages API
B) Structured results are required for prompt caching to function
C) The model cannot parse JSON larger than a few hundred tokens
D) Tool output is what fills the window fastest, so minimal payloads slow context rot, cut cost/latency, and let large results be stored externally and re-fetched by reference
Answer: D
Explanation: Tool results are the fastest-growing, lowest-signal part of the window, so returning minimal structured payloads (and storing large results externally behind a handle) is the right move. The API does not reject large outputs, the model parses large JSON fine, and caching does not depend on tool-result shape.
Tags: D5 | Intermediate | S4

## Sub-Agent Context Isolation

### Q24. S3 asks how to stop the coordinator's window from filling up as several subagents work in parallel. What is the correct design?
A) Have all subagents share one window so they can see each other's progress
B) Increase the coordinator's iteration cap to absorb the extra tokens
C) Run each subagent in an isolated context scoped to one task and return only a compact result (its findings) to the coordinator
D) Stream every subagent's tokens back into the coordinator in real time for full visibility
Answer: C
Explanation: Context isolation gives each subagent a clean, task-scoped window and returns only condensed findings across the boundary; the coordinator never ingests full transcripts. Streaming full transcripts (anti-pattern) defeats isolation and accelerates context rot; a shared window collides; and raising the iteration cap does not address the token flood.
Tags: D5 | Intermediate | S3

### Q25. In the documented orchestrator–worker (multi-agent research) pattern, what crosses the boundary from a subagent back to the lead agent?
A) Only the subagent's condensed findings/result; its exploratory context is discarded
B) Nothing — the lead re-does each subagent's work itself
C) The subagent's entire exploratory transcript, including all tool calls and dead-ends
D) The subagent's raw tool outputs, so the lead can re-verify each one
Answer: A
Explanation: Per Anthropic's "How we built our multi-agent research system," the lead decomposes the task, subagents explore in isolation, and only their condensed outputs are synthesized; exploratory context is discarded. Passing full transcripts or raw tool dumps reintroduces the rot that isolation removes, and re-doing the work defeats parallelism.
Tags: D5 | Intermediate | S3

### Q26. What reliability benefit does subagent context isolation provide beyond keeping windows small?
A) It prevents one agent's noisy tool calls and dead-ends from polluting another agent's (or the coordinator's) reasoning, and enables clean parallelization
B) It guarantees subagents never produce errors
C) It removes the need for any error handling between agents
D) It lets subagents share a single cache for lower cost
Answer: A
Explanation: Isolation stops cross-contamination of reasoning and lets windows run in parallel without colliding. It does not make subagents error-free or remove the need to surface and handle their errors, and it is about clean separate contexts, not shared caching.
Tags: D5 | Intermediate | S3

### Q27. A subagent in S3 hits a dead-end after 20 noisy exploratory tool calls. How should its result reach the coordinator?
A) Re-run the subagent in the coordinator's own window to "merge" contexts
B) Return an empty result silently so the coordinator assumes success
C) Forward all 20 tool calls and the full reasoning so the coordinator can audit them
D) Return a compact summary of the relevant finding (or a clear "no result, here's why"), discarding the exploratory noise
Answer: D
Explanation: Only the condensed, decision-relevant finding (including an honest negative result with its reason) should cross the boundary; the exploratory noise is discarded. Forwarding all 20 calls pollutes the coordinator; an empty silent result is the suppress-errors anti-pattern; and merging contexts destroys isolation.
Tags: D5 | Advanced | S3

## Reliability: Backoff, Idempotency, Graceful Degradation, stop_reason

### Q28. How should an agent decide that a tool-use loop is complete?
A) Stop when the model's output gets shorter than the previous turn
B) Stop when a fixed iteration cap is reached, regardless of state
C) Branch on the API `stop_reason` (e.g., `end_turn` vs `tool_use`) returned with each response
D) Detect the phrase "I'm done" or similar in the model's natural-language output
Answer: C
Explanation: Completion is determined by `stop_reason` (`end_turn`, `tool_use`, `max_tokens`, `pause_turn`, etc.), not by parsing prose. Reading "I'm done" from text (anti-pattern 1) is unreliable; an iteration cap is only a safety backstop (anti-pattern 2); and output length says nothing about completion.
Tags: D5 | Foundational | S1

### Q29. Why is an arbitrary iteration cap the wrong PRIMARY stopping mechanism for an agent loop?
A) Caps cause the model to refuse to use tools
B) The cap is a safety backstop, not the control; the loop should terminate on the API `stop_reason`, with the cap only preventing runaway cost
C) Caps are billed per iteration even when unused
D) Iteration caps are never useful and should be removed entirely
Answer: B
Explanation: The cap is a backstop against runaway loops, while the actual control is `stop_reason`. Caps are still worth keeping (so removing them entirely is wrong), they do not change tool-use behavior, and unused iterations are not billed.
Tags: D5 | Intermediate | S3

### Q30. Which retry strategy is correct for transient, rate-limit, and 5xx errors from the API?
A) Exponential backoff with jitter, respecting any retry guidance the API returns, up to a sensible cap
B) Immediate fixed-interval retries with no limit
C) Retry once, then crash the whole pipeline if it fails again
D) Switch to a different model provider on the first error
Answer: A
Explanation: Exponential backoff with jitter (and honoring server retry hints) handles transient/rate-limit/5xx errors without thundering-herd retries. Fixed immediate retries hammer the API; crashing after one retry is brittle; and silently switching providers (a known anti-pattern) breaks task context and masks the real issue.
Tags: D5 | Foundational | S5

### Q31. An S1 support agent's "create ticket" tool call is retried after a timeout. What prevents a duplicate ticket from being created?
A) Waiting longer before retrying so the first call surely finished
B) Adding a prompt instruction telling the model "do not create duplicates"
C) Making the operation idempotent — use an idempotency key or natural key so a retried write does not double-create
D) Lowering the retry count to one
Answer: C
Explanation: Idempotency (idempotency keys or natural keys) makes a retried create safe regardless of whether the first call landed. A lower retry count still allows one duplicate; prompt instructions are non-deterministic enforcement (anti-pattern 3); and waiting longer does not guarantee the first call's outcome is known.
Tags: D5 | Intermediate | S1

### Q32. For S5's many-file CI/CD review (large, latency-tolerant workload), which API is the right tool?
A) A single Messages call containing every file concatenated
B) Streaming responses for every file simultaneously
C) Synchronous Messages calls fired in a tight loop, one per file
D) The Message Batches API — submit a batch, poll for completion, and reconcile results
Answer: D
Explanation: The Batch API is built for large, latency-tolerant workloads: submit, poll, reconcile, instead of hammering synchronous calls. A tight synchronous loop invites rate limits; mass simultaneous streaming overwhelms limits and complicates reconciliation; and concatenating every file into one call floods the window and causes context rot.
Tags: D5 | Intermediate | S5

### Q33. A dependency a tool relies on is down. What is the correct graceful-degradation behavior?
A) Degrade to a reduced-but-correct mode — return clearly labeled partial results, fall back to a cheaper path, or escalate — rather than crashing or fabricating
B) Retry forever with no backoff until the dependency returns
C) Fabricate a plausible result so the agent can keep going
D) Return an empty success so downstream steps proceed normally
Answer: A
Explanation: Graceful degradation returns a reduced-but-correct outcome (labeled partial, cheaper fallback, or escalation). Fabrication produces silent wrong answers; an empty "success" is the suppress-errors anti-pattern (anti-pattern 7); and unbounded retries with no backoff hammer the system and never degrade.
Tags: D5 | Intermediate | S3

### Q34. Which `stop_reason` value tells you the model halted because it wants to call a tool, so your loop should execute the tool and continue?
A) `end_turn`
B) `max_tokens`
C) `stop_sequence`
D) `tool_use`
Answer: D
Explanation: `tool_use` signals the model paused to request a tool call; the loop runs the tool, appends the result, and continues. `end_turn` means the model finished its reply, `max_tokens` means output was truncated by the cap, and `stop_sequence` means a configured stop string was hit — none of which mean "run a tool."
Tags: D5 | Foundational | S1

### Q35. After a turn ends, your code reads `stop_reason: "max_tokens"`. What is the correct interpretation and action?
A) The task is fully complete; record success
B) A tool call is pending and must be executed
C) The output was truncated by the `max_tokens` cap; the response is partial, so handle continuation or raise the limit — do not treat it as a finished answer
D) The model refused the request for safety reasons
Answer: C
Explanation: `max_tokens` means generation hit the output cap and the response is incomplete; treating it as a finished, correct answer is a reliability bug. It is not a completion signal, not a refusal, and not a pending tool call (`tool_use`).
Tags: D5 | Intermediate | S5

## Surfacing Errors vs Suppressing Them

### Q36. A tool is written as `except: return []`. Why is this the single most-tested reliability anti-pattern, and what is the fix?
A) Catch the exception and return a generic "something went wrong" string
B) It is wrong only because empty lists waste tokens
C) It silently swallows the failure so the model cannot retry, escalate, or adapt; return a structured error with the failure reason (root cause visible to the model) instead
D) It is fine because an empty list is a safe default
Answer: C
Explanation: Returning `[]` on exception (anti-pattern 7) makes failure indistinguishable from a real empty result and blinds the model. The fix is a structured error surfacing what failed, why, and what to try next. Returning a generic "something went wrong" string is the related anti-pattern 6 — it hides diagnostic context just as badly.
Tags: D5 | Intermediate | S3

### Q37. Why is a generic "an error occurred" tool response harmful to an agent's reliability?
A) It hides the diagnostic context the model needs to recover, so the model cannot choose the right corrective action (retry, escalate, adapt)
B) It is only a problem for human operators, not the model
C) It triggers an automatic provider switch
D) It uses too many tokens compared to an empty result
Answer: A
Explanation: Generic error strings (anti-pattern 6) strip the root-cause detail the model needs to recover. Surface what failed, why, and a suggested next step. It is not primarily a token issue, does not switch providers, and harms the model's recovery just as much as a human's.
Tags: D5 | Intermediate | S1

### Q38. A retrieval tool gets a 403 (access denied) for a resource. How should it report this to the model?
A) Return a structured error distinguishing the access failure from an empty result, including the reason, so the model can escalate or request access rather than assume the resource is empty
B) Silently skip the resource and continue
C) Return an empty result set, identical to "no matching records found"
D) Throw the exception so the whole agent crashes
Answer: A
Explanation: An access failure and a genuine empty result are different states; returning an empty set identical to "no matching records found" lets the model wrongly conclude "nothing exists." Surface the 403 with its reason so the model can escalate or request access. Silently skipping suppresses the error; crashing the whole agent is not graceful degradation.
Tags: D5 | Advanced | S4

### Q39. In S3, a subagent fails partway through. What should the coordinator receive?
A) The subagent's full crash transcript streamed verbatim into the coordinator window
B) Nothing, so the coordinator assumes that line of inquiry yielded no results
C) A surfaced, structured failure (which subagent, what failed, partial findings if any) so the coordinator can retry, reassign, or degrade gracefully
D) A fabricated successful result to keep the pipeline moving
Answer: C
Explanation: Failures must be surfaced with diagnostic context so the coordinator can recover (retry, reassign, or degrade). Returning nothing is the suppress-errors anti-pattern; streaming the full crash transcript pollutes the coordinator window (defeats isolation); and fabricating success is dishonest and corrupts downstream synthesis.
Tags: D5 | Advanced | S3

## Per-Category Evaluation Metrics

### Q40. An S6 extraction report shows "94% overall accuracy — ship it." Why is this insufficient, and what should you do?
A) 94% is below any reasonable bar, so reject it on the number alone
B) Accuracy is the wrong metric; only latency matters for shipping
C) Aggregate accuracy can mask a category failing badly; break accuracy down per document type and gate the release on the worst slice, not the mean
D) Re-run the eval until the number reaches 99% before shipping
Answer: C
Explanation: Aggregate-only metrics (anti-pattern 10) hide a failing slice behind a healthy average; evaluate per document type / category and gate on the worst slice. The problem is not the specific number, latency is not a substitute for correctness, and re-running until a target appears is gaming, not measuring.
Tags: D5 | Intermediate | S6

### Q41. Why must per-category evaluation be paired with surfacing (not suppressing) errors?
A) Suppressing errors improves per-category precision
B) Surfacing errors makes metrics less accurate
C) They are unrelated concerns
D) Per-category metrics can only exist if failures are recorded rather than swallowed; a tool that returns empty-on-error erases the very data the slice metrics need
Answer: D
Explanation: Per-slice metrics require that failures are logged, not hidden; `except: return []` erases the failures the metrics would have counted, inflating apparent accuracy. Surfacing errors makes metrics truthful, and suppression corrupts (not improves) per-category precision.
Tags: D5 | Advanced | S5

### Q42. In S5's CI/CD review pipeline, how should results be reported so a regression in one area is visible immediately?
A) Pass/fail and accuracy broken down per check type and per file class, so a regression in one category surfaces immediately
B) A single overall pass-rate number for the whole run
C) The average review latency across all files
D) Only the count of files reviewed
Answer: A
Explanation: Reporting per check type and per file class makes a category regression visible right away, instead of being masked by the aggregate. A single overall number hides slices (anti-pattern 10); a file count and average latency say nothing about correctness per category.
Tags: D5 | Intermediate | S5

### Q43. Which evaluation practice best protects against a model that is excellent on common cases but fails on a rare, high-stakes document type?
A) Increase the sample size until the rare type's effect on the mean disappears
B) Track precision/recall per slice and gate releases on the worst-performing slice rather than the average
C) Report only the mean accuracy across all document types
D) Weight the rare type out of the metric because it is uncommon
Answer: B
Explanation: Gating on the worst slice catches a failing high-stakes category that the mean would hide. Reporting only the mean is the masking anti-pattern; weighting the rare type out of the metric deliberately hides the failure; and growing the sample to dilute its effect on the mean is the same masking error at scale.
Tags: D5 | Advanced | S6

## Mixed and Cross-Cutting Reliability Scenarios

### Q44. An S5 batch job of 5,000 file reviews returns with 60 individual failures scattered across the results. What is the correct reconciliation behavior?
A) Discard the 60 failed items silently and ship the rest
B) Surface and record the 60 failures (with reasons), retry the transient ones with backoff, and report results per check type so the failures are not masked by the aggregate
C) Mark the batch successful because the 4,940 successes dominate
D) Re-run the entire batch from scratch to be safe
Answer: B
Explanation: Per-item failures must be surfaced, retried (transient ones, with backoff/idempotency), and reflected in per-category metrics rather than averaged away. Declaring success because most passed masks the failures; silently discarding them is the suppress-errors anti-pattern; and re-running all 5,000 wastes budget when only 60 need attention.
Tags: D5 | Advanced | S5

### Q45. A coordinator compacts its history but the next step fails because a file path established 30 turns ago was dropped from the summary. What is the root-cause fix?
A) Disable compaction entirely and accept context overflow
B) Re-derive the file path by asking the model to guess from context
C) Raise `max_tokens` so the summary can be longer
D) Make compaction state-preserving — explicitly retain identifiers like file paths/IDs, decisions, and constraints — and externalize critical state to notes so it survives compaction
Answer: D
Explanation: The failure is lossy compaction dropping a load-bearing ID; the fix is state-preserving compaction plus externalized notes for critical state. Disabling compaction reintroduces overflow; a bigger `max_tokens` does not guarantee the right facts are kept; and guessing the path is unreliable and risks acting on a wrong value.
Tags: D5 | Advanced | S3

### Q46. Which combination correctly pairs a context-management problem with its proper remedy?
A) "Coordinator window fills as subagents work" → subagent context isolation returning compact results; "agent loses progress on compaction" → external memory/notes
B) "Tool returns empty on error" → keep returning empty for stability
C) "94% aggregate accuracy" → ship without per-slice analysis
D) "Agent degrades after many turns" → switch to a bigger model
Answer: A
Explanation: Isolation fixes coordinator window growth and external memory fixes progress loss across compaction — both are the documented remedies. A bigger model does not fix context rot; empty-on-error is itself the bug; and shipping on aggregate accuracy masks failing slices.
Tags: D5 | Intermediate | S3

### Q47. To minimize cost and latency on a multi-turn agent while keeping reliability high, which structure is correct?
A) Include the full conversation history every turn so nothing is lost
B) Move the system prompt below the latest user turn so it is always fresh
C) Disable caching and compaction so behavior is fully deterministic
D) Keep a stable cached prefix (system + tools), append new user/tool content below the cache breakpoint, periodically compact the middle, and return lean tool outputs
Answer: D
Explanation: A cached stable prefix, appended volatile tail, compacted middle, and lean tool outputs optimize cost, latency, and reliability together. Full history every turn causes rot and cost; disabling caching/compaction wastes money and overflows; and moving the system prompt below the user turn destroys the cacheable prefix.
Tags: D5 | Advanced | S1

### Q48. An architect echoes the current goal, active constraints, and key IDs into a compact "state" block each turn instead of relying on long history. Why is this a good practice?
A) It is unnecessary because the model always recalls earlier turns perfectly
B) It guarantees the model will never make an error
C) It replaces the need for any external memory or compaction
D) It keeps critical state high-signal and front-of-mind without forcing the model to re-derive it from a long, rot-prone history
Answer: D
Explanation: A compact state block carries forward exactly what the next step needs, reducing reliance on long history that the model may attend to poorly. The model does not recall long histories perfectly, the state block complements (not replaces) external memory/compaction, and no technique guarantees zero errors.
Tags: D5 | Intermediate | S3

### Q49. Across compaction, retries, and even crashes, what makes an agent's state durable?
A) The conversation transcript held in memory
B) A higher retry count
C) A larger context window
D) State persisted to external storage (notes, task list, memory tool) that the agent reloads selectively
Answer: D
Explanation: Externalized state is what survives compaction, retries, and crashes; the in-memory transcript does not. A bigger window still gets compacted or lost on crash, and more retries do not preserve state — they just repeat operations (which is why idempotency matters).
Tags: D5 | Intermediate | S3

### Q50. A tool returns a 200 KB raw JSON dump every call, and the agent degrades after a few tool calls. What is the correct remediation?
A) Increase the context window and keep returning the full dump
B) Make the tool return a lean, structured payload (only needed fields/IDs); store the full result externally and pass a reference the agent can re-fetch on demand
C) Tell the model in the prompt to "ignore irrelevant fields"
D) Compress the JSON with gzip before adding it to the window
Answer: B
Explanation: Tool output is the fastest-growing, lowest-signal window content; returning minimal payloads and externalizing the large result behind a handle fixes the rot at its source. A bigger window only delays the problem; gzip does not reduce token count in the window (the model still sees text); and prompting the model to ignore fields is non-deterministic and still pays the token cost.
Tags: D5 | Advanced | S4

### Q51. Why does respecting the API's returned retry guidance (e.g., a retry-after hint) matter alongside exponential backoff?
A) It forces the request to a different region automatically
B) It does not matter; client-side backoff fully replaces server guidance
C) The server can tell you how long to wait (e.g., on rate limits); honoring it avoids wasted retries and reduces the chance of repeated throttling
D) It disables jitter, which is otherwise harmful
Answer: C
Explanation: Honoring server retry hints aligns your backoff with the server's actual readiness, avoiding wasted attempts and repeated throttling. Client backoff alone may retry too soon, there is no automatic region switch, and jitter remains useful (it is not disabled by honoring hints).
Tags: D5 | Intermediate | S5

### Q52. Which scenario pairing correctly maps Domain 5 sub-topics to their anchoring scenarios?
A) Subagent isolation and coordinator compaction → S6; Batch API and per-check metrics → S2
B) Prompt caching → S6 only; token budgeting → S1 only
C) Subagent isolation, coordinator compaction, and graceful degradation → S3; Batch API, retries/idempotency, and per-check metrics → S5
D) Per-document-type accuracy → S5 only; idempotency → S3 only
Answer: C
Explanation: D5's S3 anchors cover subagent isolation, coordinator compaction, and graceful degradation; S5 anchors cover Batch API, retries/backoff/idempotency, and per-check-type/per-file-class metrics. The other pairings scramble the documented mapping (per-document-type accuracy is most associated with S6 extraction, for example).
Tags: D5 | Foundational | S5

### Q53. An agent infers it has finished a task because the model wrote "All steps are complete." What is wrong, and what is the deterministic alternative?
A) Nothing is wrong; trusting the model's self-report is standard
B) The agent should stop only when the iteration cap is reached
C) Parsing prose for completion is unreliable (anti-pattern 1); branch on `stop_reason` and verify required state (e.g., expected outputs written) before declaring completion
D) The agent should ask the model to rate its own confidence and stop above a threshold
Answer: C
Explanation: Completion must come from `stop_reason` plus a deterministic state check, not from prose. Trusting self-reported text (anti-pattern 1) and self-reported confidence (anti-pattern 4) are both unreliable signals, and an iteration cap is a backstop (anti-pattern 2), not the completion control.
Tags: D5 | Intermediate | S3

### Q54. A second model is asked to review the first model's output, but both run in the same session sharing the full reasoning history. Why might per-category review quality suffer, and what is the context-management fix?
A) The reviewer inherits the author's reasoning bias (same-session self-review, anti-pattern 9); run the review in a fresh, isolated context and report results per check type so masked failures surface
B) The reviewer should simply be given a higher `max_tokens`
C) Sharing history is ideal because the reviewer has all context
D) The reviewer should trust the author's self-reported confidence scores
Answer: A
Explanation: Same-session self-review lets the reviewer inherit the author's bias; a fresh, isolated context plus per-check-type reporting yields independent, per-slice signal. A bigger `max_tokens` does not remove the bias, and relying on self-reported confidence (anti-pattern 4) is exactly the kind of soft signal Domain 5 warns against.
Tags: D5 | Advanced | S5

### Q55. An architect wants to cache a stable 30 KB tool schema and a separate 80 KB static knowledge document, where the document is occasionally edited but the tools never change. How should breakpoints be placed?
A) No breakpoints; the API caches large blocks automatically
B) One breakpoint before the tools, caching nothing useful
C) One breakpoint after the document, so both the tools and document are cached together
D) One breakpoint after the tools and a second after the document, so editing the document invalidates only its segment while the tools cache survives
Answer: D
Explanation: Using two breakpoints lets segments that change at different rates be cached independently, so a document edit does not invalidate the unchanged tools cache. A single combined breakpoint after the document means any document edit re-encodes the tools too; caching is opt-in (not automatic); and a breakpoint before the tools caches nothing reusable.
Tags: D5 | Advanced | S5

## Answer Key (quick reference)

Q1 D · Q2 D · Q3 C · Q4 C · Q5 A · Q6 B · Q7 B · Q8 D · Q9 D · Q10 D · Q11 D · Q12 C · Q13 D · Q14 C · Q15 A · Q16 D · Q17 D · Q18 D · Q19 D · Q20 C · Q21 D · Q22 A · Q23 D · Q24 C · Q25 A · Q26 A · Q27 D · Q28 C · Q29 B · Q30 A · Q31 C · Q32 D · Q33 A · Q34 D · Q35 C · Q36 C · Q37 A · Q38 A · Q39 C · Q40 C · Q41 D · Q42 A · Q43 B · Q44 B · Q45 D · Q46 A · Q47 D · Q48 D · Q49 D · Q50 B · Q51 C · Q52 C · Q53 C · Q54 A · Q55 D
