# 07 — Production Systems Design

> Goal: design a memory service that serves millions of tenants at a p99 you can defend in a review,
> at a cost you can defend to finance, and that you can migrate without downtime.
>
> This is the chapter where the "80% distributed systems" claim from the README gets cashed. Every
> number below was produced by running something — PostgreSQL 16.14, Python 3.11 — and the outputs
> are pasted verbatim.

---

## 7.0 In plain words

### The library counter

Picture a library with one counter. A reader walks up and asks a question. Behind the counter you
have **120 milliseconds** before the reader gets bored — that is your whole slice of a two-second
answer.

In those 120ms you must:

1. decide whether this question even needs the archive (many do not),
2. look in the card catalogue (lexical) *and* the "similar books" shelf (vector),
3. maybe walk two shelves over to a related section (graph),
4. throw away the junk (rerank),
5. and hand over a single page — not a trolley — because the reader can only read one page before
   answering.

Now make it **ten million readers**, each with their own private archive, and a rule that reader A
must never, under any circumstance, see a page from reader B's archive. That is the memory service.

### The naive version

Here is what almost everyone ships first, and it is a reasonable thing to ship first:

```python
# the 20-line memory service
def handle_turn(user_id: str, message: str) -> str:
    q = embed(message)                                     # 1 embedding call
    rows = db.execute("""
        SELECT text FROM memories
        WHERE user_id = %s
        ORDER BY embedding <=> %s
        LIMIT 20
    """, (user_id, q))                                     # 1 vector query
    context = "\n".join(r.text for r in rows)
    answer = llm(f"<memory>\n{context}\n</memory>\n\nUser: {message}")
    facts = llm(EXTRACT_PROMPT.format(turn=message))       # inline extraction
    for f in facts:
        db.execute("INSERT INTO memories (user_id, text, embedding) VALUES (%s,%s,%s)",
                   (user_id, f, embed(f)))
    return answer
```

It works. It works for months. Then the arithmetic arrives.

### The arithmetic that kills it

Four independent numbers, each verified later in this chapter:

**1. Latency.** The inline extraction call is 1–3 seconds and it is on the response path. Your
"2 second to first token" target is gone before you even count retrieval. Then `ORDER BY embedding
<=> %s` without an ANN index is a sequential scan: at 20,000 memories that is a 20,000-vector dot
product per turn, per user.

**2. Tail latency compounds in a way people guess wrong.** Simulating a seven-stage pipeline
(§7.4), the sum of the per-stage p99s is **76.0 ms** but the actual p99 of the total is **56.9 ms** —
you *over*-budget by 34% if you add p99s. And the opposite mistake is worse: fan out to 10 parallel
arms and the chance that at least one of them lands past its own p99 is **9.6%**, so your "p99"
becomes something closer to a p90 experience.

**3. Memory.** 4 billion memories × 1024 dimensions × 4 bytes is **16.38 TB** of raw float32
vectors, plus 0.77 TB of HNSW link structure. No single machine holds that. But the per-tenant slice
is **837 KB** at the median, and the *hot* set — 10% DAU × 200 memories — is **858 GB**, which is
eight commodity machines. The global number is a trap; the tenant-scoped number is the real problem.

**4. Money.** At 1M MAU, the LLM calls you worry about (extraction: **$12,800/mo** on the Batch API)
are dwarfed by the tokens you inject without thinking about (**$144,000/mo** at a 3k-token memory
block). Halving the block saves **$72,000/mo** — more than the entire write path. Full derivation in
§7.8.

**5. Isolation.** And the one that ends careers: the `WHERE user_id = %s` is your only tenant
boundary, and it is enforced by a human remembering to type it. §7.3 shows a real Postgres session
where row-level security was enabled, the policy was correct, and the table owner **still saw every
tenant's rows**.

### What fixes what

| Failure of the naive version | Fixed by |
|---|---|
| Extraction on the response path | §7.7 async write path, per-tenant ordered queue |
| Sequential scan per turn | §7.2 storage topology, ANN index, partition-per-tenant-hash |
| "p99 = sum of p99s" budgeting | §7.4 verified percentile composition |
| A slow arm blows the whole budget | §7.4 deadline propagation (and the bug in the obvious implementation) |
| 16 TB of vectors | §7.2 quantisation, §7.5 sharding and hot-set sizing |
| One whale tenant ruins everyone's p99 | §7.3 working-set cap, §7.5 shard-balance arithmetic |
| Cross-tenant leak | §7.3 RLS *plus* FORCE *plus* a non-superuser role *plus* a CI test |
| Cost dominated by injected tokens | §7.8 cost model, §7.10 citation-rate instrument |
| "Recall got worse after we changed embedding models" | §7.9 dual-write / dual-read migration runbook |
| "It forgot what I said two minutes ago" | §7.7 freshness SLO and the queueing arithmetic behind it |

**The one-sentence takeaway:** memory is a tenant-scoped, read-heavy, latency-critical service whose
cost is dominated by the tokens it injects and whose worst failure is a missing WHERE clause — design
for those four facts and ignore everything the vector-database marketing is about.

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

Here is that difference in bytes. Same workload, three ways of counting it:

```
4B memories, float32 1024d: vectors 16.38 TB + hnsw links 0.77 TB = 17.15 TB
4B memories, int8    1024d: vectors  4.10 TB + hnsw links 0.77 TB =  4.86 TB
4B memories, float32  384d: vectors  6.14 TB + hnsw links 0.77 TB =  6.91 TB

one p99 tenant (20k memories), float32 1024d: 85.8 MB
one p99 tenant (20k memories), int8    1024d: 24.3 MB
one p50 tenant (200 memories), float32 1024d: 837.5 KB

hot set = 10% DAU x 200 mem = 200M vectors -> 858 GB float32, 243 GB int8
```

(HNSW link overhead estimated as `n × 2m × 4 bytes × 1.5` for `m = 16` — layer-0 neighbours at 4-byte
ids plus ~50% for upper layers. The pgvector index build documentation describes the same
`m`/`ef_construction` structure, [pgvector README](https://github.com/pgvector/pgvector).)

Read those three blocks in order and the architecture writes itself: **17 TB is an infrastructure
project; 858 GB is a cache; 837 KB is nothing at all.** Your job is to keep the system operating on
the third number as often as possible.

---

## 7.2 Storage topology

### Start here: one Postgres

```sql
CREATE TABLE memories (
  id           UUID NOT NULL,
  tenant_id    UUID NOT NULL,
  namespace    TEXT NOT NULL,            -- 'user' | 'agent' | 'org' | 'task'
  category     TEXT NOT NULL,
  text         TEXT NOT NULL,
  embedding    vector(1024),
  model_id     TEXT NOT NULL,            -- which embedding model produced it (§7.9)
  tsv          tsvector GENERATED ALWAYS AS (to_tsvector('english', text)) STORED,
  valid_from   TIMESTAMPTZ NOT NULL,
  valid_to     TIMESTAMPTZ,
  expired_at   TIMESTAMPTZ,
  confidence   REAL NOT NULL,
  schema_version INT NOT NULL DEFAULT 1, -- §7.9
  metadata     JSONB NOT NULL DEFAULT '{}',
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  last_accessed_at TIMESTAMPTZ,
  PRIMARY KEY (tenant_id, id)            -- tenant first: partition key must be in the PK
) PARTITION BY HASH (tenant_id);

CREATE INDEX ON memories USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 200);
CREATE INDEX ON memories USING gin (tsv);
CREATE INDEX ON memories (tenant_id, namespace, valid_to)
  WHERE valid_to IS NULL AND expired_at IS NULL;
```

Note `PRIMARY KEY (tenant_id, id)`. Postgres requires the partition key to be part of every unique
constraint on a partitioned table; if you write `PRIMARY KEY (id)` the `CREATE TABLE` fails outright.
Putting `tenant_id` first is also what you want for locality.

Why this is the right starting point and not a compromise:

- **Transactions across facts, vectors, and graph edges.** A supersession (ch 04/05) must atomically
  close one row and open another. With a separate vector DB this is a distributed write with no
  transaction, and you will have dangling vectors. This alone justifies the choice.
- **Filters and vectors in one engine.** Your queries are always filtered by tenant; splitting stores
  means either over-fetching then filtering (slow, and can return empty — see ch 03 on filtered ANN)
  or a two-phase query.
- **One system to operate, back up, and restore.** Point-in-time recovery for free.
- **Hash partitioning by tenant** keeps each partition's index manageable and gives you a natural
  sharding key later (§7.5).

### When to graduate, and to what

| Signal | Move to |
|---|---|
| Vector index no longer fits in RAM across partitions | Dedicated vector store, or quantise first (usually enough) |
| Multi-hop traversal is the dominant query and CTEs are too slow | Graph DB for the graph tier only; keep facts in PG (ch 05.4) |
| Need sub-10ms p99 on a small hot set | Redis/in-memory tier in front, PG as source of truth |
| > 100k QPS reads | Read replicas + per-tenant caching, then shard |

**Quantise before you migrate.** From the table in §7.1: int8 takes the same corpus from 17.15 TB to
4.86 TB — a **3.5× reduction including link overhead** (4× on the vectors themselves) — for roughly a
point of recall, recoverable by reranking the top-k against full-precision vectors. Migrating to a
new datastore takes a quarter. Quantisation takes an afternoon. Do the afternoon first. This is the
most common premature-migration mistake in this space.

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
  vectors — the *result*. See §7.6.
- **Postgres:** source of truth for facts, vectors, provenance.
- **Graph tier:** only if chapter 05 justified it, and only for the entity graph.
- **Object store:** raw episodes beyond N days, and archived cold memories. Cheap, retrievable on
  demand, and where the bulk of your bytes should live. Chapter 05.5's "non-lossy episodes" property
  is what makes object storage sufficient here: episodes are append-only and re-derivable.

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

### The RLS experiment, run for real

Everyone's first answer is "use row-level security". Correct, and insufficient in ways you cannot
guess. The following was run against PostgreSQL 16.14 with the exact policy most blog posts give you.

Setup: three rows, two tenants (alice has 2, bob has 1), a login role `mem_app`, and:

```sql
ALTER TABLE memories ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON memories
  USING (tenant_id = current_setting('app.tenant_id')::uuid);
SET app.tenant_id = '11111111-...-111111111111';   -- alice
SELECT count(*) FROM memories;                     -- deliberately no WHERE clause
```

Actual session output:

```
=== TEST 1: table OWNER (superuser), policy enabled, tenant = alice ===
 rows_visible_to_owner
-----------------------
                     3          ← ALL THREE. Bob's row included.

=== TEST 2: same query as the non-owner app role ===
 rows_visible_to_app
---------------------
                   2          ← correct

=== TEST 3: FORCE ROW LEVEL SECURITY, then re-run as owner ===
 rows_visible_to_owner_forced
------------------------------
                            3          ← STILL all three
```

**Stop at TEST 1.** RLS was enabled, the policy was right, the GUC was set — and the query returned
another tenant's private row. Two separate bypasses are in play, and PostgreSQL documents both:
the **table owner** bypasses its own row security unless you `FORCE` it, and a **superuser** (or any
role with `BYPASSRLS`) bypasses it unconditionally, `FORCE` or not
([PostgreSQL 16 docs, §5.9 Row Security Policies](https://www.postgresql.org/docs/16/ddl-rowsecurity.html)).
TEST 3 is the superuser case: `FORCE` did nothing because the role was a superuser.

Run the same pair as a **non-superuser owner** and you can see `FORCE` doing its job:

```
=== non-superuser OWNER, RLS ENABLED but not FORCED ===
 visible
---------
       2          ← owner bypass: sees both tenants

=== same role after FORCE ROW LEVEL SECURITY ===
 visible
---------
       1          ← correct
```

So the rule, stated precisely: **RLS protects you only if the connecting role is (a) not a superuser,
(b) has no `BYPASSRLS`, and (c) either does not own the table or the table is `FORCE`d.** Most
development happens as the superuser that created the database, which means **your local test of RLS
will pass while proving nothing.** That is the trap.

### The second bug: the unset GUC

Connection poolers hand you a connection that some other request used. What happens if
`app.tenant_id` is missing or blank?

```
=== TEST 4: GUC not set at all (new session simulation) ===
ERROR:  invalid input syntax for type uuid: ""

=== TEST 5: GUC set to empty string (pooled connection reset) ===
ERROR:  invalid input syntax for type uuid: ""
```

An error is *safe* — it fails closed, no rows leak — but it is a 500 on a user request, and note that
`RESET app.tenant_id` leaves the setting as `''`, not as "unset". The fail-closed-and-quiet version:

```sql
CREATE POLICY tenant_isolation ON memories
  USING (tenant_id = NULLIF(current_setting('app.tenant_id', true), '')::uuid);
```

`current_setting(..., true)` returns NULL instead of erroring when the setting is missing; `NULLIF`
maps the empty string to NULL; and `tenant_id = NULL` is NULL, which the policy treats as "not
visible". Verified:

```
=== TEST 6: fail-closed policy variant using NULLIF + missing_ok ===
 rows_when_guc_missing
-----------------------
                     0          ← no rows, no error

 rows_for_bob
--------------
            1          ← and bob still sees exactly his own
```

Writes are covered too — for `INSERT`, Postgres applies the `USING` expression as the `WITH CHECK`
expression when you do not give one, so a compromised or buggy service cannot *plant* a row in
another tenant:

```
=== TEST 7: can the app write a row belonging to another tenant? ===
ERROR:  new row violates row-level security policy for table "memories"
```

That last one matters more than it looks. Chapter 09's memory-poisoning threat model assumes an
attacker who can cause writes; a `WITH CHECK` policy turns "write into someone else's memory" from an
application bug into a database error.

### Does RLS break partition pruning?

Legitimate worry: the policy predicate is `current_setting(...)`, not a literal, so can the planner
still skip the other 63 partitions? Run it — 8 hash partitions, 20,000 rows, query as the app role
with **no WHERE clause at all**:

```
=== as app role: RLS predicate only, no explicit WHERE ===
 Aggregate (actual rows=1 loops=1)
   ->  Append (actual rows=1 loops=1)
         Subplans Removed: 7
         ->  Seq Scan on mem_part_3 (actual rows=1 loops=1)
               Filter: (tenant_id = (NULLIF(current_setting('app.tenant_id', true), ''))::uuid)
               Rows Removed by Filter: 2472
```

`Subplans Removed: 7` is **runtime partition pruning**: `current_setting` is STABLE, so the value is
known at execution start and seven of eight partitions are never touched. RLS costs you nothing
structurally here. (The residual `Rows Removed by Filter: 2472` is the other tenants inside the same
hash partition — that is what the `(tenant_id, ...)` index is for in a real table.)

### Defence in depth, the full list

1. **RLS with `FORCE`, and an application role that is not a superuser and does not own the table.**
   Test it as that role or you have tested nothing (TEST 1).
2. **Fail-closed policy expression** — `current_setting(..., true)` + `NULLIF` (TEST 6).
3. **No raw query interface in the service.** All access goes through a repository layer that takes
   `tenant_id` as a required, non-defaulted first argument, and that sets the GUC in the same
   transaction as the query.
4. **A CI test that proves it**, run against every new query path:

```python
def test_no_query_path_crosses_tenants(pg, repo_methods):
    """Every repository method, called with tenant A's id, must never return a row
    whose tenant_id is B — even for methods that forget the WHERE clause."""
    a, b = seed_tenant(pg, "alice", n=50), seed_tenant(pg, "bob", n=50)
    with pg.connect(role="mem_app") as conn:            # NOT the owner, NOT a superuser
        conn.execute("SET LOCAL app.tenant_id = %s", (a,))
        for name, method in repo_methods.items():
            rows = method(conn, tenant_id=a, **sample_args(name))
            leaked = [r for r in rows if r["tenant_id"] != a]
            assert not leaked, f"{name} leaked {len(leaked)} rows from another tenant"

def test_rls_is_actually_enforced(pg):
    """Guards against the TEST 1 failure: RLS silently disabled by role privileges."""
    row = pg.one("""SELECT relrowsecurity, relforcerowsecurity
                    FROM pg_class WHERE relname = 'memories'""")
    assert row["relrowsecurity"] and row["relforcerowsecurity"]
    app = pg.one("SELECT usesuper, userepl FROM pg_user WHERE usename = 'mem_app'")
    assert not app["usesuper"]
    assert not pg.one("""SELECT rolbypassrls FROM pg_roles
                         WHERE rolname = 'mem_app'""")["rolbypassrls"]
```

The second test is the one nobody writes and the one that would have caught TEST 1. Assert the
*configuration*, not just the behaviour, because the behaviour is correct right up until someone
changes which role the service connects as.

5. **`SET LOCAL`, not `SET`.** `SET LOCAL` is transaction-scoped, so a pooled connection cannot carry
   tenant A's GUC into tenant B's next request. This single keyword is the difference between a
   correct pooled deployment and an intermittent cross-tenant leak that reproduces once a week.
6. **Audit sampling in production:** log a hash of `(query_tenant, returned_row_tenants)` on a sample
   of requests and alert on any mismatch.

I have seen cross-tenant memory leaks in production more than once. They are catastrophic — it is not
a stale cache, it is one user's private facts shown to another. Over-engineer this.

### The noisy tenant

The p99 tenant with 500k memories will blow your per-query latency budget, dominate your cache, and
make your background jobs time out. Handle explicitly:

- **Cap the working set** used per query (e.g. only search memories in the top 50k by retention
  score; archive the rest). With HNSW the search cost grows roughly logarithmically, but the *index
  build* and the memory footprint do not — 500k × 4KB is 2 GB of one tenant sitting in your buffer
  cache.
- **Separate queue lanes** for background jobs by tenant size, so a whale's consolidation job does not
  starve 10,000 small tenants (§7.7).
- **Per-tenant rate limits and quotas**, surfaced in the product ("memory is full, review your saved
  items") rather than silently degrading.

---

## 7.4 Latency budget, and why adding p99s is wrong

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

That table is how everyone writes a latency budget, and the arithmetic in it is wrong in two
directions at once. Both are worth understanding, because they push in opposite directions and teams
routinely make one mistake while congratulating themselves on avoiding the other.

### Mistake 1: p99(A + B) ≠ p99(A) + p99(B)

Take seven serial stages with plausible `(p50, p99)` pairs, model each as lognormal, draw 200,000
samples, and measure the composition:

```
stages (p50, p99 in ms):
  gate (1,3)  cache (1,4)  vector (6,15)  lexical (4,8)
  fusion (0.5,1)  rerank (18,40)  assembly (2,5)

sum of per-stage p99  = 76.0 ms
p50 of the sum        = 33.8 ms
p99 of the sum        = 56.9 ms
p999 of the sum       = 68.8 ms
```

Summing p99s gives 76.0 ms; the true p99 is **56.9 ms**. You over-budgeted by 34%, because a request
is very unlikely to be unlucky in all seven stages at once. The real p99 of the total is driven by the
one or two stages with the fattest tails — here, `rerank`.

The practical consequence: **budget the tail to the dominant stage, not to the sum.** If you allocate
by summing p99s you will cut a feature you did not need to cut, and the feature you cut will be a
cheap one (the gate, the lexical arm) while the actual tail sits in the reranker.

Caveat, and it is a real one: the simulation assumes **independence**. When stages contend for the
same resource — one Postgres, one connection pool, one GPU — they correlate, and correlated stages
compose closer to the naive sum. If your stages share a bottleneck, measure the end-to-end
distribution directly. Never infer it.

### Mistake 2: fan-out makes the tail worse, not better

Parallelism converts *serial addition* into *maximum*, which is a huge win on the median and a
quieter loss on the tail. With independent arms:

```
vector p99 14.9 | lexical p99 8.0 | max(both) p99 14.9

   2 arms: P(at least one arm past its own p99) = 2.0%
   3 arms: P(at least one arm past its own p99) = 3.0%
   5 arms: P(at least one arm past its own p99) = 4.9%
  10 arms: P(at least one arm past its own p99) = 9.6%
```

With 10 parallel arms, 9.6% of requests contain a straggler past its own p99. If your fan-out has no
deadline, **your p90 now looks like your slowest arm's p99.** This is the tail-at-scale effect from
Dean & Barroso's "The Tail at Scale"
([CACM 56(2), 2013](https://research.google/pubs/the-tail-at-scale/)) reduced to a single line of
arithmetic: `1 − 0.99^n`.

The fix is not fewer arms. It is that **every fan-out needs a deadline and a defined degraded
result** — which brings us to the bug.

### The deadline bug in the obvious implementation

Here is the retrieval function as it is usually written, and as it appeared in the first draft of
this chapter:

```python
async def retrieve(query, tenant, budget_ms=120):
    deadline = time.monotonic() + budget_ms / 1000
    async with asyncio.TaskGroup() as tg:                    # <-- the bug
        dense   = tg.create_task(vector_search(query, tenant))
        lexical = tg.create_task(bm25_search(query, tenant))
        graph   = tg.create_task(graph_search(query, tenant))
    results = fuse([t.result() for t in (dense, lexical, graph) if t.done()])
    if time.monotonic() < deadline - 0.05:
        results = await rerank(query, results)
    return results[:8]
```

It reads correctly. It enforces nothing. `asyncio.TaskGroup.__aexit__` **waits for every child task
to finish** before the `async with` block exits, so `budget_ms` is decorative and the `if t.done()`
filter is always true for all three tasks. Running it with a graph arm that takes 900 ms:

```
draft  TaskGroup: 901 ms elapsed, 3 arms used
fixed  wait()   : 121 ms elapsed, 2 arms used
```

**901 ms against a 120 ms budget**, with the deadline check sitting right there in the source. This is
the single most common way a latency budget becomes fiction: the code that is supposed to enforce it
structurally cannot. The corrected version:

```python
async def retrieve(query, tenant, budget_ms=120, rerank_reserve_ms=45):
    t0 = time.monotonic()
    tasks = {
        asyncio.create_task(vector_search(query, tenant), name="dense"),
        asyncio.create_task(bm25_search(query, tenant),   name="lexical"),
        asyncio.create_task(graph_search(query, tenant),  name="graph"),
    }
    fanout_budget = (budget_ms - rerank_reserve_ms) / 1000
    done, pending = await asyncio.wait(tasks, timeout=fanout_budget)

    for p in pending:                                   # abandon stragglers, do not await them
        p.cancel()
        metrics.increment("memory.arm.timeout", tags={"arm": p.get_name()})
    asyncio.gather(*pending, return_exceptions=True)    # reap cancellations in the background

    results = fuse([t.result() for t in done if not t.exception()])
    if not results:
        return []                                       # degraded: no memory beats a slow answer

    remaining = budget_ms / 1000 - (time.monotonic() - t0)
    if remaining > rerank_reserve_ms / 1000:
        try:
            results = await asyncio.wait_for(rerank(query, results), timeout=remaining)
        except asyncio.TimeoutError:
            metrics.increment("memory.rerank.timeout")  # fall through with fused order
    return results[:8]
```

Four properties that the first version lacked and that you should check in any review:

1. **The fan-out has a timeout**, and the timeout reserves budget for the stage *after* it.
2. **Stragglers are cancelled, not awaited.** Cancelling and then `await`ing the gather inline would
   reintroduce the original bug if the arm swallows `CancelledError`.
3. **Every degradation is counted.** `memory.arm.timeout` by arm name tells you which arm to fix;
   without it, silent degradation looks like a mysterious quality regression.
4. **Empty is an acceptable answer.** Memory is an *enhancement*. A slightly worse memory context is
   far better than a slow response, and no memory context is better than a late one.

### What the budget implies

- **An LLM query-rewrite call does not fit.** 200 ms serial into a 120 ms budget is not a tuning
  problem. Options: run it in parallel with a no-rewrite retrieval and use whichever returns useful
  results; use a tiny distilled model; or rewrite only when a heuristic detects unresolved references
  ("it", "that one", "the same thing").
- **Cross-encoder reranking is a third of your budget and the whole tail.** Justify it with
  measurements (ch 03, ch 08). Sometimes a feature-based reranker gets most of the gain for 5 ms.
- **Rank your skip order in advance.** Mine: graph arm → rerank → lexical arm → vector arm. Write it
  down before the incident, because during the incident you will skip whatever is easiest to skip.

---

## 7.5 Capacity: the plan for the §7.1 workload

Little's law does most of the work here: `concurrency = arrival rate × service time`.

```
 50,000 qps x 25ms service = 1,250 in-flight requests
 50,000 qps x 60ms service = 3,000 in-flight requests
120,000 qps x 25ms service = 3,000 in-flight requests
```

Read that second line as the operational lesson: **a 2.4× regression in mean service time costs the
same capacity as a 2.4× traffic spike.** Latency work *is* capacity work, and it is usually cheaper.

**Database connections.** Of the 25 ms, roughly 12 ms is time inside Postgres:

```
DB-bound portion: 50,000 qps x 12ms = 600 busy connections needed
  at 100 conns/instance -> 6 pooler-backed PG instances minimum
```

600 *busy* connections, not 600 configured connections. Postgres degrades badly past a few hundred
active backends, so this is a transaction-pooling requirement (PgBouncer or equivalent), not a
`max_connections` setting. Note the interaction with §7.3: transaction pooling is exactly why the GUC
must be set with `SET LOCAL` inside the transaction.

**Shards.** How many, and does the heavy tail wreck the balance? Simulate 1M tenants drawn from a
lognormal fitted to `p50 = 200, p99 = 20,000`, capped at the observed max of 500k, and hash them into
shards:

```
sampled p50=199 p99=19991 max=500000 mean=1407
    8 shards: mean 175.83M rows, max 179.58M, max/mean = 1.021
   64 shards: mean  21.98M rows, max  23.81M, max/mean = 1.083
  256 shards: mean   5.49M rows, max   7.38M, max/mean = 1.344
```

Three things fall out, and the third is the useful one:

1. **Hash-by-tenant balances well when tenants-per-shard is large.** At 8 shards the worst shard is
   2% above the mean. Heavy tails average out when you have 125,000 tenants per shard.
2. **Imbalance grows as you add shards.** At 256 shards the worst shard is **34% above the mean**,
   because a single 500k-row whale is now a visible fraction of a 5.5M-row shard. The law of large
   numbers is doing the balancing and you are taking it away.
3. **Therefore: size shards by tenant count, not by row count.** The failure mode of "just add more
   shards" is that the whales start to matter, and you end up needing per-tenant placement — a tenant
   directory — which is a much bigger operational commitment than hash partitioning.

The practical plan for this workload: **64 logical shards** (hash partitions), which is 22M rows each
and 8% worst-case imbalance, physically co-located as 8 partitions per instance across 8 instances.
Logical shard count is fixed at design time; physical placement is what you rebalance. Moving a
logical shard is a `pg_dump`/restore of partition, not a re-hash of the world.

**What to hold in RAM.** The hot set is 858 GB at float32, 243 GB at int8 (§7.1). Spread across 8
instances that is 107 GB or 30 GB each — the int8 case fits comfortably in a commodity 128 GB box
alongside the heap. This is the single calculation that decides whether you need a dedicated vector
store, and for this workload the answer is no.

---

## 7.6 Caching

Three cache layers, each with a different invalidation story.

**L1 — Assembled context, per (tenant, intent-class).** Cache the rendered memory block, not the raw
rows. Key on a tenant memory version stamp so any write invalidates cleanly:

```python
key = f"mem:{tenant_id}:{intent_class}:{tenant_memory_version}"
```

Bump `tenant_memory_version` on every write to that tenant. Simple, correct, no fine-grained
invalidation logic to get wrong. Hit rates of 40–60% are achievable *within a session*, because the
same intent classes recur — and note the interaction with §7.7: a chatty write path invalidates L1
constantly, so batching extraction to session end (ch 04.8) is a *cache* optimisation as well as a
cost one.

**L2 — Embeddings.** Query embeddings for repeated queries; document embeddings are stored anyway.
Cheap and high-hit. At 120M retrievals/mo the embedding line is only $29 (§7.8), so L2 is a latency
optimisation (one less network round trip), not a cost one. Be honest about which you are buying.

**L3 — Provider prefix cache.** Not yours, but you influence it — see chapter 02 on ordering the
prompt from stable to volatile. Getting L1 and the prefix cache to agree (same memory block → same
tokens → same prefix) is a compounding win, and with cached input discounted up to 90%
([OpenAI pricing](https://developers.openai.com/api/docs/pricing), retrieved 2026-09-12) it is a
direct line item, not a nicety. A memory block that reshuffles every turn destroys the prefix cache
for *everything after it in the prompt*, which is why memory goes early and stable.

**Anti-pattern:** caching by semantic similarity of the query ("this query is 0.95 similar to a cached
one, reuse it"). It sounds clever, it silently serves wrong results, and the failures are very hard to
debug — the answer is plausible, just about a different question. Do not.

---

## 7.7 The write path at scale

```
extraction event ──► queue (partitioned by tenant_id) ──► workers ──► PG
                                                            │
                                                            └──► outbox ──► index updates
```

Requirements:

- **Ordering per tenant.** Partition the queue by `tenant_id` so a tenant's jobs are serial. Global
  ordering is unnecessary and expensive; per-tenant ordering is required, because supersession
  (ch 04.5) is order-dependent: process "moved to Mumbai" before "moved to Pune" and you end up
  believing Pune.
- **Idempotency.** Key on `(session_id, turn_index, extractor_version)`. Retries must be no-ops.
- **Transactional outbox** for index updates so you never have a fact committed without its vector,
  or vice versa. If you are in one Postgres, this is just one transaction — another point for
  starting there.
- **Dead letter queue with alerting.** Extraction failures are usually LLM API failures and come in
  bursts (ch 05.8: the provider quota, not the database, is your throughput ceiling).
- **Backpressure.** When the queue lags, degrade to cheaper extraction (smaller model, fewer
  categories) rather than dropping. Memory freshness degrading beats memory loss.

### The freshness SLO, and the queueing arithmetic behind it

Define it explicitly: **"a fact stated in a session is queryable in a later session within 5 minutes,
p99."** Then notice that this SLO is a statement about queue utilisation, not about worker speed.

For a single queue at utilisation ρ with mean service time S, the standard M/M/1 waiting time is
`Wq = ρ/(1−ρ) × S`. With extraction batches taking S = 3 s:

```
ρ = 0.50  →  Wq =  1.00 × 3s =    3 s
ρ = 0.80  →  Wq =  4.00 × 3s =   12 s
ρ = 0.90  →  Wq =  9.00 × 3s =   27 s
ρ = 0.95  →  Wq = 19.00 × 3s =   57 s
ρ = 0.99  →  Wq = 99.00 × 3s =  297 s   ← 4 min 57 s: the 5-minute SLO, gone
```

The shape is the point. **Between 90% and 99% utilisation, your freshness lag grows 11×** while your
dashboards show "workers busy, throughput fine". Capacity planning for the write path is therefore
not "can we keep up" — at ρ = 0.99 you are keeping up perfectly — it is "what utilisation keeps the
tail inside the SLO". Provision to ρ ≈ 0.7 and autoscale on **queue lag**, never on CPU.

Sizing for this workload: 5,000 writes/s ÷ 10 exchanges per batched call = 500 calls/s, × 3 s per call
= **1,500 concurrent extraction workers** at ρ = 1.0, so ~2,100 at ρ = 0.7. These are IO-bound
coroutines waiting on an LLM, not CPU-bound processes; the real constraint is the provider's
concurrency limit (ch 05.8's `SEMAPHORE_LIMIT` problem at a different scale).

`time_between(episode_ts, fact_indexed_ts)` is your key write-path metric and the one that correlates
with user-perceived "it forgot".

---

## 7.8 Cost model

Build this spreadsheet before you build the system. Rates below are `gpt-4.1-mini` at **$0.40/1M
input, $1.60/1M output** and `text-embedding-3-small` at **$0.02/1M**, with the Batch API at 50% of
standard ([OpenAI pricing](https://developers.openai.com/api/docs/pricing), retrieved 2026-09-12 —
treat as a snapshot, re-derive with your own rates).

Workload: 1M MAU × 20 sessions/mo × 10 exchanges = **200M exchanges/mo**, 60M new memories/mo, 60% of
turns pass the retrieval gate → 120M retrievals/mo.

```
WRITES
  extraction   20M calls x (2k in + 300 out)      $25,600/mo   →  $12,800 on Batch API
  embeddings   60M new memories x 40 tok              $48/mo
  reconcile    15M LLM adjudications (1.2k+150)   $10,800/mo
  write total (batched)                           $23,648/mo

READS
  query embeddings   120M x 12 tok                    $29/mo
  injected tokens    120M x 3,000 tok            $144,000/mo
  injected tokens    120M x 1,500 tok             $72,000/mo
  injected tokens    120M x   800 tok             $38,400/mo
  rerank  7.2B pairs / 400 pairs/s = 5,000 GPU-h  $10,000/mo at $2/GPU-hr

GRAND TOTAL (1.5k block, batched writes, self-hosted rerank)
  $23,648 + $72,029 + $10,000 = $105,677/mo  ≈  $0.106 per MAU per month

STORAGE
  60M memories/mo x (1KB text + 1KB int8 vector) = 123 GB/mo growth
  episodes (raw) — much larger; tier to object storage after 30 days
```

Five lessons fall out of this arithmetic, and they are not the ones people expect:

1. **Injected tokens dominate everything.** $144,000/mo at a 3k block versus $12,800 for all the
   extraction in the system — **11× more expensive than the LLM work you were worried about.** Cutting
   the block from 3k to 1.5k tokens saves **$72,000/mo**, which is more than the entire write path
   costs. Precision beats recall economically *and* on quality (context rot, ch 02). These incentives
   align — rare and pleasant.
2. **Embeddings are free.** $48 + $29 per month, combined, at 1M MAU. Stop optimising them. If your
   design meeting spends twenty minutes on embedding cost and none on block size, you are optimising
   the fourth decimal place.
3. **Batch everything on the write path.** The 50% Batch discount halves the largest write line for
   the price of a latency SLO you do not have (writes are async by §7.7).
4. **Self-host the reranker.** 5,000 GPU-hours is ~7 dedicated GPUs; at any per-call API rate this is
   an order of magnitude more expensive. It is a small encoder model — run it yourself.
5. **Gate aggressively.** The 60% gate rate is a multiplier on the single largest line item. Moving
   it to 40% saves another $24,000/mo *and* reduces latency *and* reduces context rot. This is the
   highest-leverage parameter in the entire system, which is why chapter 04 spends so long on it.

Report cost as **tokens per query** alongside accuracy and latency, always (standing rule 5). A memory
system's headline accuracy number is meaningless without it: 95% accuracy at 40k tokens/query loses to
90% at 5k in almost every real deployment.

---

## 7.9 Migrations you will actually do

Plan for these from day one; each one has bitten teams I know.

### Re-embedding (you will do this roughly annually)

You will change embedding models. The mistake is treating it as a backfill job; it is a **dual-read
migration** with a measurement gate.

1. **New model gets a new `model_id`.** Write new vectors *alongside* old ones in a second column or
   a sibling table. Never overwrite — you cannot roll back an overwrite.
2. **Backfill hottest-first**, not oldest-first. Order by `last_accessed_at DESC`: the tenants who
   query today get the quality improvement today, and you find out about problems on 1% of the corpus
   instead of 100%.
3. **Dual-read during transition:** query both indexes, fuse with RRF (ch 03). This is not just safety
   — it is a free A/B: log which index contributed each retrieved item.
4. **Cut over per-tenant behind a flag**, monitoring recall@k on your chapter-08 eval set per cohort.
5. **Drop old vectors only after a full soak period** (one full billing month is a good default; you
   want to have seen a month-end reporting workload).

The backfill arithmetic decides your schedule, so do it first. 4B memories at 40 tokens each is 160B
tokens; at `text-embedding-3-small` rates that is **$3,200** — trivially affordable, which surprises
people. The constraint is throughput, not money: at a sustained 10,000 embeddings/s the backfill takes
`4e9 / 1e4 = 400,000 s ≈ 4.6 days` of continuous running. Plan for a week, run it hottest-first, and
keep the dual-read path up the entire time.

**The invariant that makes this survivable:** never mix vectors from two models in one ANN index.
Cosine distance between embeddings of different models is meaningless — not "slightly worse",
*meaningless*. `model_id` on every row, one index per model, and a query path that refuses to compare
across them. The symptom when you get this wrong is a recall collapse for exactly the cohort that
finished backfilling, which looks like the new model is bad.

### Extractor version upgrade

Do *not* re-extract everything blindly — you will invalidate memories users have come to rely on, and
a user who sees a correct preference disappear does not care that your F1 went up.

Instead: run the new extractor in **shadow**, diff outputs on a sample, quantify the delta by
category, then backfill **only where the new extractor finds something the old one missed**. Keep
`extractor_version` on every row so the backfill is targetable and the rollback is
`WHERE extractor_version = 'v7'`. This is the same column chapter 04.10's scenario suite keys its
regression gate on.

### Schema evolution

Memories live for years — longer than your services, possibly longer than your company's current
architecture. Use `schema_version` on each row and support reading N−1 and N−2 forever. Never do an
in-place destructive migration on the memory table; it is user data, and unlike a cache it cannot be
recomputed from anything except the episode log (which is precisely why chapter 05.5's non-lossy
episode tier is a systems-design requirement, not a modelling nicety).

### Tenant migration (B2B) and shard rebalancing

Export/import of a tenant's full memory graph, *including provenance and episode references*. Build
this early. It is required for enterprise deals, for GDPR portability (ch 09), and — the reason it
belongs in this chapter — for your own shard rebalancing when §7.5's whale tenants make a physical
instance hot. One mechanism, three business justifications, which is the easiest funding argument you
will ever make.

---

## 7.10 Observability

Metrics that matter, grouped by what they tell you:

**Health**
- `memory.retrieval.latency` — p50/p95/p99, **tagged by stage**, plus the end-to-end distribution
  measured directly (per §7.4, you cannot compose it).
- `memory.arm.timeout{arm}` — the degradation counter from §7.4. If this is nonzero and nobody
  noticed, your quality metrics are not sensitive enough.
- `memory.write.lag` — episode timestamp → indexed timestamp, p99. Your freshness SLO (§7.7).
- `memory.queue.depth`, `memory.queue.utilisation` (autoscale on these, not CPU), DLQ rate.

**Quality**
- `memory.retrieval.empty_rate` — how often retrieval returns nothing. Sudden jumps mean a broken
  filter or a corrupted index, and it is the fastest-moving leading indicator you have.
- `memory.injected_tokens` p50/p99 — the cost driver from §7.8, on the dashboard next to accuracy.
- `memory.gate.skip_rate` — the other cost driver.
- `memory.contradiction_rate` — same subject+predicate, both valid (ch 05.2's unique index should
  make this structurally zero; if it is not, the index is missing).
- `memory.duplicate_rate` — cosine > 0.95 pairs within a tenant.
- `memory.usage_rate` — fraction of injected memories the model actually referenced (below).

**Business**
- `memory.user_correction_rate` — users saying "no, that's wrong". The best available proxy for memory
  quality and the one I would put on the team's dashboard.
- `memory.deletion_requests` — also a privacy obligation clock (ch 09).

### Measuring whether memory is used at all

The subtlest and most valuable instrument. You inject 8 memories; how many mattered?

```python
# Cheap approximation: ask the model to cite.
CITE = "When you use a fact from <user_memory>, cite its id inline like [m_812]."

def usage_rate(injected_ids: list[str], answer: str) -> float:
    cited = set(re.findall(r"\[m_(\w+)\]", answer))
    return len(cited & set(injected_ids)) / max(len(injected_ids), 1)
```

Run it on a 1% sample continuously. If your citation rate is 15%, you are injecting 85% waste —
tokens, latency, and distraction. I have seen this number below 10% in real systems. Put §7.8's
arithmetic next to it: at a 3k-token block and 15% usage, you are paying **$122,400/mo to inject text
the model ignores.** Measuring it usually leads directly to halving k with no quality loss, which is
a large, easy win that requires no new infrastructure.

Two honesty caveats, because this metric is easy to over-trust:

- **Citation is a proxy, not ground truth.** A memory can influence an answer without being cited
  (it ruled something out), and a model can cite decoratively. Calibrate quarterly against
  **counterfactual ablation**: re-run the turn with each memory removed and measure whether the answer
  changes. Expensive — k+1 generations per turn — so run it weekly on a sample of 200 turns and fit
  the cheap proxy to it.
- **Asking for citations changes the output.** It is a prompt modification, so run it on a sample
  cohort, not on everyone, and check that the cohort's quality metrics match the control.

---

## 7.11 Failure modes and the runbook

| Failure | Symptom | Immediate action | Root fix |
|---|---|---|---|
| Extraction queue saturated at ρ→1 | "It forgot what I just said"; lag 5min+ | Scale workers; degrade to cheap extractor | §7.7 autoscale on lag, provision to ρ≈0.7 |
| Fan-out has no deadline | p99 tracks the slowest arm | Cap the slow arm, ship the timeout | §7.4 `asyncio.wait(timeout=)`, not `TaskGroup` |
| Latency budget "fits" but p99 is blown | budget built by summing p99s | Measure end-to-end distribution | §7.4 composition, tail to the dominant stage |
| RLS enabled but not enforcing | Cross-tenant rows in a query with no WHERE | Kill switch on memory injection; page | §7.3 `FORCE` + non-superuser role + config assertion test |
| Pooled connection carries a stale tenant | Intermittent, unreproducible wrong-user facts | Drain pool; switch to `SET LOCAL` | §7.3 transaction-scoped GUC |
| Recall collapses for one cohort | Two embedding models in one index | Force single `model_id` per query | §7.9 dual-read, one index per model |
| Vector index bloated after mass delete | Recall drops, latency up | Rebuild index | Scheduled reindex; monitor tombstone ratio |
| Extractor prompt regression | Junk memories spike | Roll back prompt; quarantine by `extractor_version` | ch 04.10 scenario suite gates deploys |
| Hot tenant OOM | p99 spike, some queries time out | Cap working set for that tenant | §7.3 working-set cap by default |
| Adding shards made balance worse | One shard 30%+ above mean | Move physical partitions, not logical shards | §7.5 size shards by tenant count |
| Cost 3× the model | Nobody measured injected tokens | Halve k, tighten the gate | §7.8 + §7.10 `usage_rate` |
| Memory poisoning | Agent behaves oddly across sessions | Quarantine tenant's memories; snapshot rollback | ch 09 controls |

The **kill switch** deserves its own paragraph: you must be able to disable memory injection
globally, per tenant, and per category, **without a deploy**. Memory is the component most likely to
cause a user-visible privacy incident, and "we need 40 minutes to roll out a fix" is not an acceptable
answer during one. Build the flag on day one, exercise it in a game day, and make sure the degraded
path (no memory) is a tested code path rather than an untested branch — the §7.4 `return []` case.

---

## 7.12 Exercises

1. **Reproduce the RLS failure.** Create a table, enable RLS with the standard `current_setting`
   policy, and query it as the superuser that owns it. Acceptance: you see all tenants' rows. Then add
   `FORCE`, switch to a non-superuser non-owning role, and re-run. Acceptance: exactly one tenant's
   rows, and `test_rls_is_actually_enforced` passes. Finally `RESET app.tenant_id` and confirm your
   policy fails closed with **zero rows and no error**, not a 500.
2. **Break your own deadline.** Take your retrieval fan-out, inject 900 ms of artificial latency into
   one arm, and measure wall time. Acceptance: elapsed ≤ budget + 5%, the slow arm's result is absent,
   and `memory.arm.timeout{arm="graph"}` incremented by exactly 1. If elapsed ≈ 900 ms you have the
   `TaskGroup` bug from §7.4.
3. **Compose the tail yourself.** Instrument every stage of your real pipeline for a day. Compute
   (a) the sum of per-stage p99s and (b) the measured p99 of the total. Acceptance: you can state the
   ratio and say whether your stages are independent or contend for a shared resource. If (b) > (a),
   find the shared bottleneck — that is a real finding.
4. **Capacity plan.** Write the plan for the §7.1 workload: shard count, instance count, RAM for the
   hot set, connection budget, worker count at ρ = 0.7. Acceptance: every number traceable to an
   arithmetic step, and a stated answer to "what breaks first if traffic doubles?"
5. **Cost, twice.** Compute monthly cost with your own rates. Then recompute with the memory block
   halved. Acceptance: you can name the single largest line item and its share of the total. (If it
   is not injected tokens, either your block is already tight or your arithmetic is wrong.)
6. **Shard balance.** Simulate your own tenant-size distribution into 8, 64, and 256 shards.
   Acceptance: a max/mean ratio per configuration and a chosen shard count justified by it, not by
   round numbers.
7. **Citation rate.** Implement `usage_rate` on any agent you have and report the number. Then halve
   k and measure accuracy, tokens, and latency together (standing rule 5). Acceptance: a three-column
   before/after table. The expected result is "accuracy unchanged, tokens halved" — if accuracy drops,
   you have found that your reranker order is load-bearing, which is also worth knowing.
8. **Migration dry run.** Re-embed 1% of your corpus with a different model, dual-read with RRF, and
   log which index contributed each hit. Acceptance: a per-cohort recall@10 comparison, and a
   demonstration that querying across `model_id` values produces garbage — do it once so you never
   ship it.

Next: `08-evaluation.md` — how to know whether any of this actually works, and how to stop a
regression from shipping.
