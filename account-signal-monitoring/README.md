# Signal Intelligence Engine

An n8n-based system that continuously monitors multiple external signal sources for business development triggers, scores each signal against a defined ideal customer profile using retrieval-augmented generation, and routes qualified opportunities to the appropriate internal channel in near real time.

## Problem

Manual account monitoring does not scale. Relevant buying signals, funding announcements, executive hires, office expansions, and public disputes, surface across dozens of disconnected sources and are easy to miss or act on too late. This system replaces manual monitoring with an automated pipeline that ingests signals from multiple sources, evaluates each one against a structured ideal customer profile, and delivers only the signals that matter to the people who need to act on them.

## Architecture

Three independent trigger sources feed a single shared scoring engine:

1. **Flagged Account Intake** (webhook): accepts ad hoc signal submissions from internal tools or manual flags.
2. **News Monitor** (scheduled poll, 6h): queries NewsAPI against a maintained watchlist of target accounts.
3. **RSS Monitor** (scheduled poll, 6h): polls company blog and changelog feeds for the same watchlist.

All three call into a reusable sub-workflow, **Signal Scoring Engine**, which performs the following for each incoming signal:

1. Deduplication against previously processed signals, keyed on source URL.
2. Embedding generation (Gemini `gemini-embedding-001`, 768 dimensions).
3. Retrieval of the most relevant ideal customer profile sections via a Postgres RPC function performing cosine similarity search over a Supabase pgvector store.
4. Structured scoring against the retrieved profile context (Gemini, temperature 0.1, enforced response schema, reasoning field ordered before the confidence field to force explicit justification ahead of a numeric score).
5. Routing: Hot signals post immediately to a dedicated Slack channel with source attribution and profile citation. Warm and Cold signals are logged and held for daily digest delivery.

A separate scheduled workflow, **Signal Digest Delivery**, batches all undigested Warm and Cold signals once daily, posts a summary to a secondary Slack channel, and marks each included row as digested in Supabase to prevent duplicate delivery.

## Data model

Supabase tables:

- `all_signals`: every signal processed, its score, reasoning, source URL, and a digest-delivery flag.
- `watched_accounts`: companies monitored via the News source.
- `watched_feeds`: companies monitored via the RSS source.
- `icp_embeddings`: chunked and embedded sections of the ideal customer profile, used for retrieval at scoring time.

## Tool selection

**n8n** over Make or Zapier: this project required real conditional retrieval logic, a Postgres RPC call, and a shared sub-workflow callable from three independent triggers. n8n's native Code node and sub-workflow support made this straightforward without paid add-ons. Make's equivalent capability requires its paid Code app; Zapier does not support reusable sub-workflows called from multiple independent triggers in the same way.

**Gemini** over other model providers: single-provider consistency across embedding and generation simplifies credential management, cost tracking, and prompt tuning. `gemini-embedding-001` was selected over deprecated prior embedding models and configured with an explicit output dimensionality of 768 to match the vector store schema.

**Supabase with pgvector** over a dedicated vector database: the ideal customer profile corpus is small and changes infrequently. A managed Postgres instance with pgvector avoids operating a separate vector database for a corpus this size, while still supporting exact and approximate similarity search as the corpus grows.

## Error handling

- Every external API call (Gemini embedding, Gemini scoring, Supabase retrieval) has retry logic configured: up to three attempts with a fixed delay between attempts, to absorb transient rate limiting and timeouts without manual intervention.
- Every branch capable of producing zero output, a failed profile match, a duplicate signal, a non-Hot score, has an explicit terminal path. In n8n, a node producing zero output items does not propagate execution, so any branch or sub-workflow call left without an explicit catch-all will silently halt the calling loop after its first zero-output result. This is the most consequential architectural constraint in the system and is treated as a first-class design requirement, not an edge case.
- Deduplication runs before any paid API call in every trigger path, so a duplicate signal never reaches the embedding or scoring stage.

## Scope decisions

The following were deliberately left out of this system's current scope:

- **Cross-session signal decay weighting.** A signal's relevance is currently treated as static once scored. Time-based decay of older signals was scoped out because the ideal customer profile's trigger windows are already narrow enough that stale signals rarely reach a Hot score in practice; revisiting this is a matter of tuning the profile, not the pipeline.
- **Additional trigger sources beyond News, RSS, and direct intake.** The three sources in place cover the primary channels through which the target buying signals actually surface for this ICP. Additional sources (social listening, job board APIs) would extend coverage but were not required to validate the pipeline's core value.
- **Approximate nearest-neighbor indexing on the profile store.** At current corpus size, exact similarity search is faster and simpler to reason about than a tuned ivfflat index. This is revisited once the profile corpus grows past the point where exact search is no longer performant.

## Enterprise scaling

- The shared sub-workflow pattern means a new trigger source (a fourth data feed, for example) requires only a new trigger workflow that calls the existing scoring engine, not a rebuild of the scoring logic.
- Retry configuration on every external call gives the system headroom for higher-volume account lists without requiring immediate rework.
- The vector store and RPC-based retrieval pattern scale to a materially larger ideal customer profile corpus by introducing an appropriately sized ivfflat or HNSW index at that point, a change isolated entirely to the retrieval node and the underlying SQL function.
- Digest batching and the daily delivery cadence are configurable independent of the scoring pipeline, allowing delivery frequency to change without touching how signals are scored.

## Cost

Gemini API usage (embedding and scoring) and Supabase are billed on usage-based or capped free tiers at current volume. NewsAPI is used on its free tier, with a two-day lookback padding applied to its `from` parameter to avoid its free-tier restriction on near-real-time date ranges. At the account-list volumes tested in this system, all usage remains within free or minimal-cost thresholds; a production deployment monitoring a larger account list would need to budget for NewsAPI's paid tier or an alternative news API depending on polling frequency and watchlist size.

