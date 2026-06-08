# Contextual Retrieval for RAG (CCA-F Reference)

Source: Anthropic, "Introducing Contextual Retrieval" — https://www.anthropic.com/news/contextual-retrieval. Supporting docs: Anthropic prompt caching (https://platform.claude.com/docs/en/docs/build-with-claude/prompt-caching) and the Anthropic cookbook contextual-embeddings guide. This note maps the technique to exam Domain D5 (Context Management & Reliability) and to any retrieval-heavy scenario.

## What Standard RAG Does (the baseline pipeline)

Retrieval-Augmented Generation (RAG) injects external knowledge into a prompt so Claude can answer over a corpus that does not fit in (or should not be statically baked into) the context window. The classic offline pipeline is: **chunk the documents -> embed each chunk -> store vectors in a vector database -> at query time embed the query and retrieve the top-k nearest chunks -> stuff those chunks into the prompt as context -> generate the answer.** This is sometimes shortened to "chunk, embed, store, retrieve, stuff."

Chunking is necessary because documents are usually longer than what you want to retrieve at once, and because finer-grained chunks give more precise matches. Typical chunk sizes are a few hundred tokens (e.g., paragraphs or sliding windows of a few sentences), with some overlap so ideas are not severed mid-thought.

Embeddings power **semantic** retrieval: similar meaning maps to nearby vectors, so a query like "how do I cancel" can match a chunk about "terminating your subscription" even with no shared keywords. Anthropic does not ship a first-party embedding model; the docs point to third-party providers (e.g., Voyage AI) for embeddings. **Exam signal:** if an answer claims "use Anthropic's embedding endpoint," treat it with suspicion — embeddings are sourced from a provider, while Claude is used to *contextualize* chunks, not to embed them.

## The Core Problem: Context-Free Chunks Lose Meaning

When you split a document into small chunks, each chunk is embedded and stored in isolation, **stripped of the surrounding document context**. A chunk that reads "The company's revenue grew by 3% over the previous quarter" is nearly useless on its own: which company, and which quarter? A user asking "What was ACME Corp's Q2 2023 revenue growth?" may fail to retrieve that exact chunk because the chunk text never mentions ACME or Q2 2023 — that information lived in nearby chunks or the document title.

This is the central failure mode Contextual Retrieval attacks: **isolated chunks are ambiguous, so both semantic and lexical search miss them.** The fix is to add the missing context back *into the chunk itself* before indexing.

## Contextual Embeddings: Prepend a Chunk-Specific Context Blurb

Contextual Embeddings prepends a short, chunk-specific explanatory blurb to each chunk *before* embedding it. You give Claude the whole document plus the specific chunk and ask it to write 1–2 sentences (typically 50–100 tokens) that situate that chunk within the overall document. That generated context is prepended to the chunk, and the combined text is what you embed and what you index for BM25.

For the example above, Claude might generate: *"This chunk is from ACME Corp's Q2 2023 financial report; the prior quarter referenced is Q1 2023."* Prepending that to the raw sentence makes the chunk self-describing, so a query mentioning ACME and Q2 2023 now retrieves it. The contextualization is grounded in the actual document — it is not a generic summary and not invented metadata.

**Exam signal:** the right pattern is "use Claude to generate a per-chunk context blurb grounded in the full document, prepend it, then embed." Wrong answers include: embedding raw chunks with no context (the baseline), prepending the same global document summary to every chunk (not chunk-specific), or relying purely on a larger top-k to compensate (more noise, not more signal).

## Contextual BM25: Keep Lexical (Exact-Match) Retrieval and Contextualize It Too

BM25 is a lexical, keyword-ranking function (a TF-IDF-style algorithm) that excels at exact matches — error codes, identifiers, function names, SKUs, rare proper nouns. Embeddings capture meaning but can miss a literal token like `TS-999` or `getUserById`; BM25 nails those but is blind to paraphrase. Contextual Retrieval applies the same contextualization to the BM25 index: the prepended context blurb is included in the text BM25 indexes, so lexical search also benefits from the added document context.

## Hybrid Retrieval: Combine Semantic Embeddings + Lexical BM25

The recommended retrieval stage runs **both** a vector (embedding) search and a BM25 search, then merges and de-duplicates the candidate lists (rank fusion). This hybrid approach is strictly more robust than either alone: embeddings catch conceptual/paraphrased matches, BM25 catches exact-token matches, and together they cover each other's blind spots. Contextual Embeddings + Contextual BM25 together is the configuration Anthropic reports as the strong combined result.

**Exam signal:** "embeddings alone are sufficient" and "BM25 alone is sufficient" are both distractors. For corpora with codes, IDs, and exact terms, hybrid is the defensible architecture.

## Reranking: Re-Score the Merged Candidates Before Stuffing

Hybrid retrieval can surface, say, the top ~100–150 candidates — more than you want to (or can affordably) put in the prompt. A **reranker** is a separate model that takes the query and each candidate and produces a relevance score, letting you keep only the top-N (e.g., top 20) most relevant chunks to actually pass to Claude. This adds a small latency/cost step but further cuts retrieval failures by pushing the best chunks to the top and dropping near-duplicates and weak matches. Anthropic reports reranking on top of Contextual Embeddings + Contextual BM25 as the best-performing configuration in their tests.

The retrieval order to remember: **contextualize chunks (offline) -> hybrid retrieve (embeddings + BM25) -> merge/de-dup -> rerank -> keep top-N -> stuff into the prompt.**

## Use Prompt Caching to Make Contextualizing Each Chunk Cheap

The obvious objection to Contextual Embeddings is cost: you make a Claude call per chunk, and each call includes the *whole document* so the context blurb is grounded. Naively, that re-sends the full document once per chunk. **Prompt caching solves this:** load the full document once into a cached prompt prefix, then for each chunk reuse the cache and only pay full price for the small, per-chunk completion. Cache reads are billed at a steep discount versus a normal input token, so contextualizing all chunks of a document becomes inexpensive. Anthropic frames this as the key enabler that makes per-chunk contextualization economically practical (a roughly one-time, low single-digit dollar cost per million document tokens, in their writeup — treat the exact figure as approximate).

**Exam signal:** the connection "prompt caching makes Contextual Retrieval affordable by caching the document prefix across all its chunks" is a high-value, testable link between D5 and the prompt-caching topic. A wrong answer will claim contextualization is prohibitively expensive (ignoring caching) or that you must fine-tune a model to do it.

## Reported Impact (present as approximate)

In Anthropic's experiments, measured by retrieval failure rate (fraction of relevant documents not in the top-k retrieved):

- **Contextual Embeddings alone: ~35% reduction** in the top-20 retrieval failure rate versus standard embeddings.
- **Contextual Embeddings + Contextual BM25: ~49% reduction** — the lexical signal compounds the gain.
- **Adding a reranking stage on top: further reduction (Anthropic reports roughly two-thirds overall, ~67%).**

State these as approximate, directional results, not guarantees — they are corpus- and configuration-dependent. The exam-relevant takeaway is the **ordering and the why**: contextualization > +BM25 > +reranking, each layer addressing a distinct failure source (ambiguous chunks, missed exact terms, suboptimal ranking).

## When to Use RAG vs Long-Context vs Agentic Search

**Long-context (no retrieval):** if the entire relevant knowledge base fits comfortably in the context window, you can skip retrieval and load it directly — and use prompt caching to keep that large static prefix cheap across requests. Anthropic notes that for knowledge bases under roughly 200K tokens, just putting everything in the prompt can outperform a RAG pipeline and is simpler. **Exam signal:** "always build RAG" is wrong when the corpus is small enough to cache in-context.

**RAG (retrieve-then-generate):** the default once the corpus exceeds the window or you have a large/growing/changing knowledge base. Contextual Retrieval is the quality upgrade to make that RAG reliable.

**Agentic search:** instead of one offline top-k fetch, let the agent issue tools (search, file reads, MCP queries, follow-up queries) and iterate over multiple turns. This fits exploration, multi-hop questions, codebase navigation, and freshness-sensitive lookups where the right query is not known upfront. It trades latency/cost for adaptiveness. Many production systems combine them: agentic loops that call a Contextual-Retrieval-backed search tool. **Exam signal:** match the method to the corpus and the question shape — small/static -> long-context; large/static-ish factual lookup -> Contextual RAG; exploratory/multi-hop/fresh -> agentic search over a retrieval tool.

## Reliability Anti-Patterns to Recognize (D5)

- **Embedding raw, context-free chunks** and then blaming "the model" for wrong answers — the failure is in retrieval, not generation. Fix retrieval with contextualization first.
- **Cranking up top-k to brute-force recall.** Stuffing 100 chunks adds noise, dilutes attention, raises cost, and still misses chunks whose text is ambiguous. Reranking + contextualization beat a giant top-k.
- **Semantic-only retrieval on corpora full of IDs/codes.** Without BM25 you will silently miss exact-token queries. (Silent retrieval misses mirror anti-pattern #7: a confident answer over an incomplete context window looks successful but isn't.)
- **Skipping caching and concluding contextualization is "too expensive."** That is a cost-model error; the document prefix is cacheable.
- **Reporting one aggregate retrieval-accuracy number.** Evaluate per query type / per document type (ties to anti-pattern #10: aggregate metrics mask per-slice failures). Use a held-out set and measure retrieval failure rate at your real top-k.

## One-Line Summary for Recall

Standard RAG = chunk -> embed -> store -> top-k retrieve -> stuff. Contextual Retrieval improves it by using Claude (cheaply, via prompt caching) to **prepend a chunk-specific context blurb before indexing**, then doing **hybrid embeddings + BM25** retrieval, then **reranking** the merged candidates — cutting retrieval failures by roughly 35% / 49% / ~67% as you add each layer. Choose long-context for small corpora, Contextual RAG for large static knowledge bases, and agentic search for exploratory or multi-hop tasks.
