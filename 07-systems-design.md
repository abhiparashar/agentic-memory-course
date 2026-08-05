# 07 — Production Systems Design

> Goal: design a memory service that serves millions of tenants at a p99 you can defend in a review,
> at a cost you can defend to finance, and that you can migrate without downtime.
>
> This is the chapter where the "80% distributed systems" claim from the README gets cashed.

---

## 7.1 The workload, characterised

Before any architecture, characterise the load. Memory workloads have an unusual shape, and getting
this wrong is why teams over-buy infrastructure.

```
Tenants:            10M users
Memories/tenant:    p50 = 200, p95 = 3,000, p99 = 20,000, max = 500,000
Total memories:     ~4B  (but never queried globally)
Read QPS:           50k (proportional to active conversations)
Write QPS:          5k (extraction jobs, async)
Read latency SLO:   p50 < 30ms, p99 < 150ms  (part of a ~2s end-to-end budget)
Write latency SLO:  none (async) — but freshness SLO: p99 < 5 min
Access pattern:     EXTREMELY skewed. A tenant's memories are queried only by that tenant.
                    Hot set is small: DAU × recent memories.
```

Four properties dominate every design decision:

1. **Queries are always tenant-scoped.** There is no global search. This is the most important fact
   about the workload and it means **you never need a single global index.**
2. **The distribution is heavy-tailed.** Most tenants are tiny; a few are enormous. Any design that
   assumes uniform tenant size will fall over on the tail.
3. **Reads dominate and are latency-critical; writes are batchable and latency-tolerant.**
4. **The hot set is small.** Yesterday's inactive users' memories are cold. Cache and tier
   accordingly.

**The single-tenant-scope property is worth stating loudly because it changes everything.** Vector
database marketing is built around billion-scale global ANN search. Your problem is 10 million
independent 200-to-20,000-vector searches. Those are completely different systems problems, and the
second one is much easier. Do not buy a solution to the first.

---

## 7.2 Storage topology

### Start here: one Postgres

```sql
CREATE TABLE memories (
  id           UUID PRIMARY KEY,
  tenant_id    UUID NOT NULL,
  namespace    TEXT NOT NULL,            -- 'user' | 'agent' | 'org' | 'task'
  category     TEXT NOT NULL,
  text         TEXT NOT NULL,
  embedding    vector(1024),
  tsv          tsvector GENERATED ALWAYS AS (to_tsvector('english', text)) STORED,
  valid_from   TIMESTAMPTZ NOT NULL,
  valid_to     TIMESTAMPTZ,
  expired_at   TIMESTAMPTZ,
  confidence   REAL NOT NULL,
  metadata     JSONB NOT NULL DEFAULT '{}',
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  last_accessed_at TIMESTAMPTZ
) PARTITION BY HASH (tenant_id);

CREATE INDEX ON memories USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 200);
CREATE INDEX ON memories USING gin (tsv);
CREATE INDEX ON memories (tenant_id, namespace, valid_to)
  WHERE valid_to IS NULL AND expired_at IS NULL;
```

Why this is the right starting point and not a compromise:

- **Transactions across facts, vectors, and graph edges.** A supersession must atomically close one
  row and open another. With a separate vector DB this is a distributed write with no transaction,
  and you will have dangling vectors. This alone justifies the choice.
- **Filters and vectors in one engine.** Your queries are always filtered by tenant; splitting stores
  means either over-fetching then filtering (slow, and can return empty) or a two-phase query.
- **One system to operate, back up, and restore.** Point-in-time recovery for free.
- **Hash partitioning by tenant** keeps each partition's index manageable and gives you a natural
  sharding key later.

### When to graduate, and to what

| Signal | Move to |
|---|---|
| Vector index no longer fits in RAM across partitions | Dedicated vector store, or quantise first (usually enough) |
| Multi-hop traversal is the dominant query and CTEs are too slow | Graph DB for the graph tier only; keep facts in PG |
| Need sub-10ms p99 on a small hot set | Redis/in-memory tier in front, PG as source of truth |
| > 100k QPS reads | Read replicas + per-tenant caching, then shard |

**Quantise before you migrate.** int8 scalar quantisation gives a 4× reduction for ~1% recall loss
and takes an afternoon. Migrating to a new datastore takes a quarter. Do the afternoon first. This is
the most common premature-migration mistake in this space.

### The hybrid topology at scale

```
                    ┌─────────────────┐
   read path ──────►│  Memory Service │
                    └────────┬────────┘
             ┌───────────────┼────────────────┬─────────────────┐
             ▼               ▼                ▼                 ▼
      ┌────────────┐  ┌────────────┐   ┌────────────┐   ┌──────────────┐
      │ Redis      │  │ Postgres   │   │ Graph tier │   │ Object store │
      │ hot tenant │  │ facts +    │   │ (optional) │   │ cold episodes│
      │ working set│  │ vectors    │   │            │   │ + archives   │
      └────────────┘  └────────────┘   └────────────┘   └──────────────┘
```

- **Redis:** the assembled memory context per active tenant, keyed by a version stamp. Not raw
  vectors — the *result*. See 7.5.
- **Postgres:** source of truth for facts, vectors, provenance.
- **Graph tier:** only if chapter 05 justified it, and only for the entity graph.
- **Object store:** raw episodes beyond N days, and archived cold memories. Cheap, retrievable on
  demand, and where the bulk of your bytes should live.

---

## 7.3 Multi-tenancy and isolation

Three isolation models; know the trade-offs cold because this is a standard design-review question.

| Model | Isolation | Ops cost | Noisy neighbour | When |
|---|---|---|---|---|
| Shared table, `tenant_id` filter | Logical | Low | Yes | Consumer scale, 10M+ tenants |
| Schema/partition per tenant | Strong logical | Medium | Reduced | B2B, thousands of tenants |
| Database per tenant | Physical | High | No | Regulated, enterprise, dozens–hundreds |

For consumer scale you will use the shared model. Then **the tenant filter becomes a security
boundary**, and security boundaries enforced by remembering to add a WHERE clause fail eventually.
Defence in depth:

1. **Postgres row-level security**, so even a buggy query cannot cross tenants.
   ```sql
   ALTER TABLE memories ENABLE ROW LEVEL SECURITY;
   CREATE POLICY tenant_isolation ON memories
     USING (tenant_id = current_setting('app.tenant_id')::uuid);
   ```
2. **No raw query interface in the service.** All access goes through a repository layer that takes
   `tenant_id` as a required, non-defaulted first argument.
3. **A test that asserts cross-tenant reads return zero rows**, run in CI, including for every new
   query path.
4. **Audit sampling in production**: log a hash of `(query_tenant, returned_row_tenants)` on a
   sample of requests and alert on any mismatch.

I have seen cross-tenant memory leaks in production more than once. They are catastrophic —
it is not a stale cache, it is one user's private facts shown to another. Over-engineer this.

### The noisy tenant

The p99 tenant with 500k memories will:
- blow your per-query latency budget,
- dominate your cache,
- and make your background jobs time out.

Handle explicitly:
- **Cap the working set** used per query (e.g. only search memories in the top 50k by retention
  score; archive the rest).
- **Separate queue lanes** for background jobs by tenant size, so a whale's consolidation job does not
  starve 10,000 small tenants.
- **Per-tenant rate limits and quotas**, surfaced in the product ("memory is full, review your saved
  items") rather than silently degrading.

---

## 7.4 Latency budget

End-to-end interactive agent target: ~2s to first token. Memory's slice:

```
 total memory read budget:            120 ms (p99)
 ├─ gate / should-we-retrieve           2 ms   (heuristic; 15ms if a model call)
 ├─ query rewrite                       0 ms   (skip) or 200ms (LLM — see below)
 ├─ cache lookup                        2 ms
 ├─ vector search (tenant-scoped)      15 ms
 ├─ lexical search                      8 ms
 ├─ graph traversal (if enabled)       20 ms
 ├─ fusion                              1 ms
 ├─ rerank (cross-encoder, 60 cands)   40 ms
 ├─ assembly + formatting               5 ms
 └─ slack                              27 ms
```

Immediate consequences:

- **An LLM query-rewrite call does not fit.** Options: run it in parallel with a no-rewrite retrieval
  and use whichever returns useful results; use a tiny distilled model; or rewrite only when a
  heuristic detects unresolved references. Do not put a 200ms serial call in a 120ms budget.
- **Cross-encoder reranking is a third of your budget.** Justify it with measurements. Sometimes a
  cheaper feature-based reranker gets 80% of the gain for 5ms.
- **Parallelise the retrieval arms.** Vector, lexical, and graph are independent — fan out.

```python
async def retrieve(query, tenant, budget_ms=120):
    deadline = time.monotonic() + budget_ms / 1000
    async with asyncio.TaskGroup() as tg:
        dense   = tg.create_task(vector_search(query, tenant))
        lexical = tg.create_task(bm25_search(query, tenant))
        graph   = tg.create_task(graph_search(query, tenant))
    results = fuse([t.result() for t in (dense, lexical, graph) if t.done()])
    if time.monotonic() < deadline - 0.05:
        results = await rerank(query, results)          # only if budget remains
    return results[:8]
```

**Deadline propagation with graceful degradation** is the pattern: if the budget is exhausted, return
un-reranked results rather than blowing the SLO. Memory is an *enhancement*; a slightly worse memory
context is far better than a slow response. Design every stage to be skippable and rank the skip
order in advance.

---

## 7.5 Caching

Three cache layers, each with a different invalidation story.

**L1 — Assembled context, per (tenant, intent-class).** Cache the rendered memory block, not the
raw rows. Key on a tenant memory version stamp so any write invalidates cleanly:

```python
key = f"mem:{tenant_id}:{intent_class}:{tenant_memory_version}"
```

Bump `tenant_memory_version` on every write to that tenant. Simple, correct, no fine-grained
invalidation logic to get wrong. Hit rates of 40–60% are achievable within a session, because the
same intent classes recur.

**L2 — Embeddings.** Query embeddings for repeated queries; document embeddings are stored anyway.
Cheap and high-hit.

**L3 — Provider prefix cache.** Not yours, but you influence it — see chapter 02 on ordering the
prompt from stable to volatile. Getting the L1 cache and the prefix cache to agree (same memory block
→ same tokens → same prefix) is a compounding win: a stable memory block means the whole prefix stays
cached across turns.

**Anti-pattern:** caching by semantic similarity of the query ("this query is 0.95 similar to a cached
one, reuse it"). It sounds clever, it silently serves wrong results, and the failures are very hard
to debug. Do not.

---

## 7.6 The write path at scale

```
extraction event ──► queue (partitioned by tenant_id) ──► workers ──► PG
                                                            │
                                                            └──► outbox ──► index updates
```

Requirements:

- **Ordering per tenant.** Partition the queue by `tenant_id` so a tenant's jobs are serial. Global
  ordering is unnecessary and expensive; per-tenant ordering is required.
- **Idempotency.** Key on `(session_id, turn_index, extractor_version)`. Retries must be no-ops.
- **Transactional outbox** for index updates so you never have a fact committed without its vector,
  or vice versa. If you are in one Postgres, this is just one transaction — another point for
  starting there.
- **Dead letter queue with alerting.** Extraction failures are usually LLM API failures and come in
  bursts.
- **Backpressure.** When the queue lags, degrade to cheaper extraction (smaller model, fewer
  categories) rather than dropping. Memory freshness degrading beats memory loss.

### Freshness SLO

Define it explicitly: "a fact stated in a session is queryable in a later session within 5 minutes,
p99." Then monitor it. `time_between(episode_ts, fact_indexed_ts)` is your key write-path metric and
it is the one that correlates with user-perceived "it forgot".

---

## 7.7 Cost model

Build this spreadsheet before you build the system. Per 1M monthly active users, 20 sessions/month,
10 exchanges/session:

```
WRITES
  extraction: 200M exchanges/mo, batched 10/call = 20M calls
    @ ~2k in + 200 out tokens, small model      →  the dominant LLM cost
  embeddings: ~60M new memories × 1 embedding   →  small but nonzero
  reconciliation: ~60M decisions (batchable)    →  comparable to extraction

READS
  retrieval: 200M turns × (say 60% gated in)    = 120M retrievals
    vector+lexical: pure compute                →  cheap
    reranking: 120M cross-encoder calls × 60    →  significant; GPU-hours, self-host
  injected tokens: 120M × 1.5k tokens           =  180B extra input tokens/mo
                                                →  usually the single largest line item

STORAGE
  60M memories/mo × (1KB text + 1KB vector int8-quantised)  ≈ 120 GB/mo growth
  episodes (raw) — much larger; tier to object storage after 30 days
```

The lessons that fall out of this arithmetic:

1. **Injected tokens usually dominate.** Reducing the memory block from 3k to 1.5k tokens is worth
   more than any infrastructure optimisation. Precision beats recall economically *and* on quality
   (context rot). These incentives align — rare and pleasant.
2. **Batch everything on the write path.** Batch APIs, batched extraction, off-peak scheduling.
3. **Self-host the reranker.** It is a small encoder model; running it on your own GPUs is
   dramatically cheaper than per-call APIs at this volume.
4. **Gate aggressively.** Every skipped retrieval saves latency, injected tokens, and cache
   invalidation.

Report cost as **tokens per query** alongside accuracy, always. A memory system's headline accuracy
number is meaningless without it — 95% accuracy at 40k tokens/query loses to 90% at 5k in almost
every real deployment.

---

## 7.8 Migrations you will actually do

Plan for these from day one; each one has bitten teams I know.

**Re-embedding (you will do this ~annually).**
1. New model gets a new `model_id`; write new vectors alongside old ones.
2. Backfill in the background, oldest-first or hottest-first (hottest-first gets you quality sooner).
3. Dual-read: query both indexes, fuse with RRF during the transition — this also gives you a free
   A/B comparison.
4. Cut over per-tenant behind a flag; monitor recall on your eval set per cohort.
5. Drop old vectors only after a full soak period.

**Extractor version upgrade.**
Do *not* re-extract everything blindly — you will invalidate memories users have come to rely on.
Instead: run the new extractor in shadow, diff outputs on a sample, quantify the delta, then backfill
only where the new extractor finds something the old one missed. Keep `extractor_version` on rows so
you can target the backfill.

**Schema evolution.**
Memories live for years. Use a `schema_version` on each row and support reading N-1 and N-2 forever.
Never do an in-place destructive migration on the memory table; it is user data.

**Tenant migration (B2B).**
Export/import of a tenant's full memory graph, including provenance. Build this early — it is
required for enterprise deals, GDPR portability, and your own shard rebalancing.

---

## 7.9 Observability

Metrics that matter, grouped by what they tell you:

**Health**
- `memory.retrieval.latency` (p50/p95/p99, by stage)
- `memory.write.lag` — episode timestamp → indexed timestamp (your freshness SLO)
- `memory.queue.depth`, DLQ rate

**Quality**
- `memory.retrieval.empty_rate` — how often retrieval returns nothing. Sudden jumps mean a broken
  filter or index.
- `memory.injected_tokens` (p50/p99)
- `memory.gate.skip_rate`
- `memory.contradiction_rate` — same subject+predicate, both valid. Should be near zero.
- `memory.duplicate_rate` — cosine > 0.95 pairs within a tenant.
- `memory.usage_rate` — fraction of injected memories the model actually referenced (see below).

**Business**
- `memory.user_correction_rate` — users saying "no, that's wrong". The best available proxy for
  memory quality and the one I would put on the team's dashboard.
- `memory.deletion_requests`

### Measuring whether memory is used at all

The subtlest and most valuable instrument. You inject 8 memories; how many mattered?

```python
# Cheap approximation: ask the model to cite
"...When you use a fact from <user_memory>, cite its id like [m_812]."
# then parse citations from the output
```

If your citation rate is 15%, you are injecting 85% waste — tokens, latency, and distraction. I have
seen this number below 10% in real systems. Measuring it usually leads directly to cutting k in half
with no quality loss, which is a large, easy win.

A stronger offline version: counterfactual ablation. Re-run the turn with each memory removed and
measure answer change. Expensive, but run it weekly on a sample to calibrate the cheap proxy.

---

## 7.10 Failure modes and the runbook

| Failure | Symptom | Immediate action | Root fix |
|---|---|---|---|
| Extraction queue backs up | "It forgot what I just said" | Scale workers; degrade to cheap extractor | Autoscaling on lag; backpressure policy |
| Vector index corrupted after mass delete | Recall drops, latency up | Rebuild index | Scheduled reindex; monitor tombstone ratio |
| Extractor prompt regression | Junk memories spike | Roll back prompt; quarantine memories by `extractor_version` | Scenario suite gates deploys (ch. 04.10) |
| Embedding model version skew | Recall drops for a cohort | Force single model per query | `model_id` per vector; dual-serve |
| Cross-tenant leak | Wrong user's facts appear | Kill switch on memory injection; page | RLS + repository layer + CI test |
| Memory poisoning | Agent behaves oddly across sessions | Quarantine tenant's memories; snapshot rollback | Ch. 09 controls |
| Hot tenant OOM | p99 spike, some queries time out | Cap working set for that tenant | Working-set cap by default |

The **kill switch** deserves emphasis: you must be able to disable memory injection globally, per
tenant, and per category, without a deploy. Memory is the component most likely to cause a
user-visible privacy incident, and "we need 40 minutes to roll out a fix" is not an acceptable answer
in that incident. Build the flag on day one.

---

## 7.11 Exercises

1. Write the capacity plan for the workload in 7.1. Choose an architecture, justify every component,
   and compute the monthly cost. Then compute it again assuming injected tokens drop by half. This
   exercise is the entire business case for precision.
2. Implement the retrieval pipeline with deadline propagation. Inject artificial latency into one arm
   and verify graceful degradation rather than SLO violation.
3. Set up RLS in Postgres and write the CI test that proves cross-tenant isolation. Then deliberately
   introduce a query that omits the tenant filter and confirm RLS catches it.
4. Implement the citation-based usage metric on any agent you have. Report the number. Then halve k
   and measure whether answer quality changed at all.

Next: `08-evaluation.md`.
