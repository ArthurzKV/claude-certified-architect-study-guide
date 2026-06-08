# Batch, Scale, and Model Selection — Cost & Throughput Architecture (D5, S5)

This reference covers how a Claude Certified Architect chooses the right *execution mode* (synchronous, streaming, or batch), the right *model tier* (Opus, Sonnet, Haiku), and the right *latency and cost optimizations* for a workload. The exam tests these as design trade-offs anchored to scenarios — especially **S5 Claude Code for CI/CD** (multi-pass review, batch-scale evals) and **D5 Context Management & Reliability** (rate limits, token budgets, graceful degradation). Sources: Anthropic platform docs (platform.claude.com/docs), the Message Batches API docs, and the model overview pages.

## The Three Execution Modes: Synchronous, Streaming, Batch

The Messages API supports three ways to run a request, and choosing among them is a recurring exam decision. **Synchronous** (a normal blocking `messages.create` call) returns the full response in one HTTP response — simplest, lowest engineering overhead, best when a human or downstream system is waiting on the answer in real time. **Streaming** (`stream=true`, Server-Sent Events) returns tokens incrementally as they are generated — same price and same total latency to *completion*, but dramatically lower *time-to-first-token*, which is what makes chat UIs and agent loops feel responsive. **Batch** (the Message Batches API) is asynchronous: you submit many requests in one job, Anthropic processes them off the latency-critical path, and you poll for results that arrive within a target turnaround window.

The decision rule the exam wants: **is anything waiting on this response right now?** If a user or a real-time agent loop needs the answer, use synchronous (or streaming for perceived speed). If nothing is waiting and you have *volume*, use batch to cut cost.

Exam signal: A scenario describes "10,000 documents to label overnight" or "run the full regression eval suite nightly" — the correct answer is the **Batch API**, not a hand-rolled loop of synchronous calls with `sleep()` between them. A loop of synchronous calls is the distractor: it burns rate limits, costs full price, and adds fragile retry plumbing you'd have to build yourself.

## The Message Batches API — Asynchronous Bulk Processing

The **Message Batches API** lets you submit a large set of independent Messages requests as a single asynchronous job. Anthropic documents it as roughly **50% cheaper** than the equivalent synchronous calls, with results delivered within a target turnaround (commonly described as **within ~24 hours**, often much faster in practice). It is designed for **high request volumes and large total sizes** that would be impractical or rate-limit-bound to fire one at a time. See platform.claude.com/docs (Message Batches API).

Each request inside a batch carries a **`custom_id`** you assign, and the result set is keyed by that id — this is how you correlate outputs back to inputs when results come back out of order or partially. Each entry in the batch is a complete Messages request, so anything you can do synchronously (tool use, system prompts, prompt caching, extended thinking, structured outputs via tools) you can do inside a batch. Results are retrieved by polling the batch's status and then downloading the results file once it reports as ended/completed.

The canonical batch use cases — and the ones that map to exam scenarios — are **non-latency-sensitive bulk work**: dataset labeling, large-scale evaluations, offline content generation, bulk classification/extraction, and **CI multi-pass code review** where each file or each review pass is an independent request. Because batch is asynchronous, it is *wrong* for anything a human is actively waiting on (live chat, an interactive agent step, an IDE completion).

Exam signal (S5): "Your CI pipeline reviews every changed file with multiple specialized passes (security, style, correctness) on each nightly merge." The right architecture submits those passes as a **batch job** — it is cheaper, it scales to large diffs, and the nightly window is not latency-sensitive. Pairing each request's `custom_id` with the file+pass lets you reassemble structured results. The trap answer runs the review inline in the latency-critical PR path with full-price synchronous calls.

## When NOT to Use Batch

Batch is the wrong tool when **latency matters or the result feeds an interactive loop**. A customer-support agent answering a live ticket (S1), an interactive Claude Code session (S2/S4), or any step where the model's output is the input to its *own next decision* must be synchronous/streaming — you cannot pause an agent's reasoning loop for hours. Batch is also a poor fit when requests are *dependent* on each other's outputs (request N needs the answer to request N−1); batch entries are processed independently, so chained reasoning belongs in a synchronous agent loop, not a batch.

Exam signal: Watch for scenarios that *sound* bulk ("score 5,000 conversations") but are actually serving a live dashboard with sub-second SLAs — there the answer is a fast synchronous model (Haiku), not batch, because the turnaround window violates the latency requirement.

## Model Selection — Opus, Sonnet, Haiku Trade-offs

Anthropic's Claude family is tiered, and the architect's job is to **match the model to the cognitive load of the task**, not to default to the biggest model everywhere. The three tiers (see the model overview at platform.claude.com/docs):

- **Opus** — the most capable reasoning and agentic model. Use it for the *hardest* work: long-horizon agentic orchestration, complex multi-step planning, deep code reasoning, ambiguous problem-solving. Highest cost and highest latency per token. Reserve it for where its capability actually changes the outcome.
- **Sonnet** — the **balanced workhorse**. Strong reasoning and coding at substantially lower cost and latency than Opus. This is the sensible *default* for most production tasks, including the bulk of Claude Code work and most agent steps.
- **Haiku** — the **fast, cheap** tier. Ideal for high-volume, well-scoped, lower-complexity tasks: **routing, classification, extraction, simple tool calls, and triage**. Lowest latency and lowest cost, which makes it the right pick for sub-tasks fanned out at scale.

The core architectural pattern: **route cheaper models for sub-tasks.** A coordinator or front-door step may use Sonnet/Opus to plan and decompose, then dispatch the high-volume, low-judgment sub-tasks (classify this ticket's category, extract these fields, decide which queue) to **Haiku**. This "use the smallest model that meets the quality bar for each sub-task" discipline is exactly what the exam rewards.

Exam signal (D1/D5): A scenario asks you to optimize cost on a pipeline that does "intent classification → retrieval → drafting → final review." The strong answer uses **Haiku for the classification/routing step**, Sonnet for drafting, and reserves Opus only for the genuinely hard final reasoning — rather than running Opus end-to-end. Defaulting everything to Opus "to be safe" is a cost anti-pattern; defaulting everything to Haiku and missing on the hard reasoning step is a quality anti-pattern. The right answer is *tiered routing matched to per-step difficulty*.

## Embeddings for RAG — Use Voyage AI, Not a First-Party Endpoint

A frequently-tested fact: **Anthropic does not offer a first-party `/embeddings` endpoint.** For Retrieval-Augmented Generation and semantic search, Anthropic **recommends Voyage AI** as the embeddings provider (Anthropic acquired Voyage AI). So the architecture for a RAG system is: chunk and embed documents with a **Voyage AI embedding model**, store the vectors in your vector database, retrieve relevant chunks at query time, and pass them into a Claude model (Messages API) as context to generate the grounded answer.

Exam signal (D5, S4): If a scenario asks "which Anthropic API generates the embeddings for your vector store?" the answer is a **trap** — there is no Anthropic embeddings endpoint. The correct choice is to use **Voyage AI for embeddings** and Claude for generation/reasoning. Any answer claiming `client.embeddings.create(...)` on the Anthropic SDK is wrong.

## Rate Limits and Token Budgets

Production reliability depends on respecting **rate limits**, which Anthropic enforces per organization across dimensions like **requests per minute (RPM)**, **input tokens per minute (ITPM)**, and **output tokens per minute (OTPM)**, with limits varying by model and usage tier. When you exceed a limit, the API returns **HTTP 429**, and responses include rate-limit headers that tell you remaining capacity and when it resets. The correct client behavior is **exponential backoff with jitter on 429s**, honoring the reset signal — not tight-loop retrying, which makes congestion worse.

The Batch API is the *structural* answer to rate-limit pressure for bulk work: instead of hammering the synchronous endpoint and tripping 429s, you hand Anthropic the whole workload and let it schedule processing within your batch capacity. This is why "we keep hitting 429s on our nightly labeling job" is almost always solved by **moving the job to batch**, not by adding more retry layers.

Exam signal (D5): A scenario with a bursty bulk workload tripping 429s wants two correct moves — (1) implement **backoff/jitter** for transient synchronous traffic, and (2) **migrate bulk, non-urgent work to the Batch API** to take it off the rate-limited synchronous path. The distractor adds an arbitrary fixed `sleep()` or silently drops requests on 429 (a silent-error anti-pattern).

## Latency Optimization Toolkit

When a workload *is* latency-sensitive, the architect has a specific toolkit, and the exam expects you to reach for the right lever:

- **Prompt caching** — cache large, stable prefixes (system prompts, tool definitions, big reference documents, long codebase context) so they aren't reprocessed every call. Cache reads are billed at a steep discount and reduce both cost and time-to-first-token. This is the single highest-leverage optimization for agent loops and Claude Code sessions that reuse the same large context across many turns. See platform.claude.com/docs (prompt caching).
- **Smaller models** — drop to **Haiku** (or Sonnet from Opus) for the latency-critical step when the task complexity allows it. A faster model is often a bigger latency win than micro-optimizing prompts.
- **Streaming** — use SSE to cut perceived latency (time-to-first-token) for any user-facing output, even when total generation time is unchanged.
- **Parallel tool calls** — allow the model to request multiple independent tool calls in a single turn and execute them concurrently in your tool runner, instead of serializing round-trips. This collapses multi-tool latency dramatically for agents (S1, S3, S4).
- **Tight context / smaller prompts** — fewer input tokens means faster prefill; trim retrieved context to what's relevant rather than stuffing the window.

Exam signal (D5): "Our agent feels slow because each turn re-sends a 40k-token system prompt and runs tools one at a time." The right fixes are **prompt caching the stable prefix** and **enabling parallel tool execution** — plus dropping latency-critical sub-steps to a smaller model. The trap answer is "increase the iteration cap" or "ask the model in the prompt to be faster," neither of which addresses the actual latency source.

## Putting It Together — A Cost/Latency Decision Checklist

For any scenario, reason in this order: (1) **Is anything waiting on the result?** No → consider **Batch** (≈50% cheaper, async). Yes → synchronous/streaming. (2) **What is the cognitive load per step?** Route **Haiku** for routing/classification/extraction, **Sonnet** as the default workhorse, **Opus** only for the hardest reasoning/agentic steps. (3) **Is the same large context reused across calls?** → turn on **prompt caching**. (4) **Are tool calls independent?** → run them **in parallel**. (5) **Are you tripping 429s on bulk work?** → move it to **Batch** and add **backoff/jitter** on the remaining synchronous traffic. (6) **Need embeddings for RAG?** → **Voyage AI**, because there is no first-party Anthropic embeddings endpoint.

Exam signal (S5 capstone): The strongest CI/CD answer typically combines several of these at once — **Batch API** for the nightly multi-pass review (cost + scale), **structured outputs via tool_use** so each pass returns machine-parseable results keyed by `custom_id`, **Haiku/Sonnet** routing so cheap passes don't pay Opus prices, and **per-slice evaluation** (per file type / per check) rather than a single aggregate pass-rate that hides where review quality is failing.
