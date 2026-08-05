# 05 — Temporal and Graph Memory

> Goal: represent a world that changes. Answer "what is true now", "what was true then", and
> "how did we get here" — plus multi-hop questions that flat retrieval cannot reach.

---

## 5.1 Why flat memory fails on time

Take a user whose history contains:

```
2025-01-10  "I work at Acme as a backend engineer"
2025-11-02  "Got promoted to tech lead"
2026-03-15  "Starting at Beta Corp next month"
```

Ask: *"Where does the user work?"*

Flat vector retrieval returns all three (all similar to the query), the model sees three
contradictory statements, and it will either pick one arbitrarily, hedge, or hallucinate a merge.
Adding a recency boost helps until you ask *"What was their role at Acme?"*, where the newest fact is
the wrong one.

The fix is not better retrieval. It is a better **data model**: facts need time.

---

## 5.2 Bi-temporal modelling

Two independent time axes per fact. This is a well-established idea in databases (SQL:2011 has it
natively) and it is exactly the right tool here.

| Axis | Name | Meaning | Used for |
|---|---|---|---|
| World time | `valid_from` / `valid_to` | when the fact was true in reality | "where did they live in 2024?" |
| System time | `recorded_at` / `expired_at` | when *we* knew it | "why did the agent say that on March 3?" audit |

Graphiti (the engine behind Zep) is built on exactly this: every edge carries validity intervals for
when the fact held, plus separate timestamps for when the system created or invalidated it — four
timestamps in total. When new information conflicts with existing knowledge, the old edge's validity
window is closed rather than deleted, so historical accuracy is preserved without recomputation.

### Why you need both, concretely

- **World time alone** cannot answer "the agent recommended a vegetarian place on March 3 — did it
  know about the dietary preference then?" You need to know when you *learned* it.
- **System time alone** cannot handle retroactive information: on 2026-06-01 the user says "I moved to
  Mumbai back in January". World-time `valid_from` is January; system time is June. Both are true and
  both matter.

This retroactive case is common and it is the clearest argument for bi-temporality. Systems using
only `created_at` get it wrong every time.

### Schema

```sql
CREATE TABLE facts (
  id            UUID PRIMARY KEY,
  tenant_id     UUID NOT NULL,
  subject_id    UUID NOT NULL,           -- entity
  predicate     TEXT NOT NULL,           -- 'works_at', 'lives_in', 'prefers'
  object_id     UUID,                    -- entity, if the object is one
  object_value  TEXT,                    -- literal otherwise
  -- world time
  valid_from    TIMESTAMPTZ NOT NULL,
  valid_to      TIMESTAMPTZ,             -- NULL = still true
  -- system time
  recorded_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  expired_at    TIMESTAMPTZ,             -- NULL = still believed
  -- provenance & belief
  source_episode_ids UUID[] NOT NULL,
  confidence    REAL NOT NULL,
  extractor_version TEXT NOT NULL
);

CREATE INDEX ON facts (tenant_id, subject_id, predicate, valid_from DESC);
CREATE INDEX ON facts (tenant_id, valid_to) WHERE valid_to IS NULL AND expired_at IS NULL;
```

The partial index on currently-valid facts is the one your hot path uses; keep it small and it stays
in cache.

### The three canonical queries

```sql
-- 1. What is true now?
SELECT * FROM facts
WHERE tenant_id = $1 AND subject_id = $2
  AND valid_to IS NULL AND expired_at IS NULL;

-- 2. What was true at time T? (world-time travel)
SELECT * FROM facts
WHERE tenant_id = $1 AND subject_id = $2
  AND valid_from <= $T AND (valid_to IS NULL OR valid_to > $T)
  AND expired_at IS NULL;

-- 3. What did we BELIEVE at time T? (system-time travel — for audit/debug)
SELECT * FROM facts
WHERE tenant_id = $1 AND subject_id = $2
  AND recorded_at <= $T AND (expired_at IS NULL OR expired_at > $T);
```

Query 3 is your incident-response tool. When a user complains "the agent said something wrong last
Tuesday", you reconstruct exactly the belief state it had. I have never regretted building this; I
have regretted not having it.

---

## 5.3 Extracting temporal information

Users state time in three ways, and you must handle all three:

- **Absolute:** "on March 4th 2025" → parse directly.
- **Relative:** "two weeks ago", "last month", "since I graduated" → resolve against the message
  timestamp (`t_ref`).
- **Implicit:** "I work at Acme" → `valid_from` = now, `valid_to` = NULL. Present tense implies
  current validity.

Graphiti resolves relative expressions against a reference timestamp from the episode. Do the same:
always pass the message timestamp into the extraction prompt.

```python
TEMPORAL = """Given a claim and the message timestamp, determine validity.

Claim: {claim}
Message sent at: {t_ref}

Return JSON:
{{"valid_from": ISO8601|null, "valid_to": ISO8601|null, "precision": "day|month|year|unknown",
  "tense": "past|present|future|habitual"}}

Rules:
- Present tense with no marker: valid_from = t_ref, valid_to = null.
- Past tense ("used to", "worked at"): valid_to should be set; estimate valid_from if stated.
- Future ("starting next month"): valid_from in the future — the fact is NOT yet true.
- Habitual ("I usually run mornings"): valid_from = t_ref, valid_to = null, tense = habitual.
- Relative expressions resolve against t_ref.
"""
```

The **future** case trips people up. "I'm starting at Beta Corp next month" must not immediately
override the current employer. Store it with a future `valid_from` and let the "what is true now"
query naturally exclude it until the date arrives. This is a genuinely nice property of the model:
correctness over time comes for free from the query, with no cron job flipping flags.

`precision` matters for conflict resolution: "in 2024" (year precision) should not override "on
2024-03-15" (day precision) for the same attribute.

---

## 5.4 When to use a graph

Be honest about this, because graph databases carry real operational cost.

**A graph earns its place when:**
- Your queries are **multi-hop**: "who at my company works on the project my manager mentioned?"
- **Relationships between entities are themselves data** you query, not just attributes.
- You need **traversal-based retrieval** — start from an entity and walk outward — because
  similarity search cannot express "connected to".
- You have **many entity types** with rich interconnection (enterprise, CRM, org knowledge).

**A graph is overkill when:**
- Your memory is 95% "facts about one user" — that is a table with a `subject_id` column.
- Your queries are single-hop lookups by attribute.
- You have < 100k facts per tenant.

Practical middle path I recommend: **model your data as a graph (subject-predicate-object triples)
but store it in Postgres.** You get the semantics without operating Neo4j. Add a real graph database
when you can demonstrate a query pattern that recursive CTEs cannot serve at acceptable latency.
Recursive CTEs handle 2–3 hops on modest data perfectly well.

```sql
-- 2-hop traversal in plain Postgres
WITH RECURSIVE walk(id, depth, path) AS (
  SELECT $start_entity, 0, ARRAY[$start_entity]
  UNION ALL
  SELECT CASE WHEN f.subject_id = w.id THEN f.object_id ELSE f.subject_id END,
         w.depth + 1, w.path || f.id
  FROM walk w
  JOIN facts f ON (f.subject_id = w.id OR f.object_id = w.id)
  WHERE w.depth < 2
    AND f.valid_to IS NULL AND f.expired_at IS NULL
    AND NOT (f.id = ANY(w.path))
)
SELECT DISTINCT id, depth FROM walk WHERE depth > 0;
```

---

## 5.5 The Graphiti/Zep architecture, explained

Worth studying in detail because it is the most fully articulated production design in the open
literature. Zep's paper (arXiv 2501.13956) is short and readable; the code is open source.

### Three-tier subgraph hierarchy

```
┌─ Episode subgraph ─────────────────────────────────────┐
│  Raw events: messages, documents, structured records   │
│  Non-lossy. Each node timestamped with the real event  │
│  time. This is your source of truth.                   │
└──────────────────────┬─────────────────────────────────┘
                       │ extraction
┌──────────────────────▼─────────────────────────────────┐
│─ Semantic entity subgraph ─────────────────────────────│
│  Entity nodes + fact edges. Edges carry bi-temporal    │
│  validity. Bidirectional links back to source episodes.│
└──────────────────────┬─────────────────────────────────┘
                       │ clustering (label propagation)
┌──────────────────────▼─────────────────────────────────┐
│─ Community subgraph ───────────────────────────────────│
│  Clusters of densely connected entities, each with a   │
│  generated summary. Enables high-level retrieval.      │
└────────────────────────────────────────────────────────┘
```

Design properties worth stealing wholesale, whatever you build:

1. **Non-lossy episodes.** Facts are derived, never the only copy. You can always re-derive with a
   better extractor. This single property makes every future model upgrade tractable.
2. **Bidirectional indices** between episodes and derived facts. Facts → episodes gives you citation
   and audit; episodes → facts gives you fast incremental updates.
3. **Bi-temporal edges with invalidation, not deletion.** Discussed above.
4. **Community summaries** as an abstraction tier. This is the "semantic memory from episodic memory"
   move: raw recall at the bottom, abstraction at the top, and retrieval can enter at either level.
   It parallels the community-summary idea in graph-RAG systems generally.

### Retrieval in this architecture

Composed from three search methods, then fused and reranked:

```
query
  ├─► cosine similarity over edge/node embeddings
  ├─► BM25 full-text over fact text
  └─► breadth-first graph traversal from seed nodes
          │
          ▼  fuse (RRF / MMR / optional cross-encoder)
          ▼  context constructor: render facts WITH their validity ranges
```

Note the last step. Rendering `works_at(User, Acme) [2025-01-10 → 2026-04-01]` rather than
`User works at Acme` is what lets the model reason about currency instead of guessing. It is a
formatting decision with outsized quality impact — cheap to adopt even in a non-graph system.

---

## 5.6 Entity resolution in a graph, done carefully

The graph amplifies both the value and the risk of entity resolution. A correct merge unlocks
multi-hop reasoning; an incorrect merge silently fuses two people's lives.

Pipeline:

```python
def resolve_entity(mention: str, ctx: str, tenant: str) -> EntityDecision:
    # 1. Blocking — cheap candidate generation
    cands  = kg.alias_lookup(mention, tenant)                     # exact/normalised alias hits
    cands += kg.vector_search(embed(f"{mention} | {ctx}"), tenant, limit=20)

    # 2. Feature scoring
    scored = []
    for c in dedupe(cands):
        f = {
            "name_sim":   jaro_winkler(normalise(mention), normalise(c.name)),
            "embed_sim":  cosine(embed(ctx), c.context_embedding),
            "type_match": float(infer_type(mention, ctx) == c.type),
            "neighbour_overlap": jaccard(neighbours(c), neighbours_in_context(ctx, tenant)),
            "recency":    recency_weight(c.last_seen),
        }
        scored.append((c, model.score(f)))

    best, s = max(scored, key=lambda t: t[1], default=(None, 0.0))
    if s > 0.90:  return EntityDecision("MERGE", best.id)
    if s < 0.55:  return EntityDecision("CREATE_NEW", None)
    return EntityDecision("DEFER", best.id)     # queue for LLM adjudication / human review
```

`neighbour_overlap` is the highest-signal feature and the one people omit. Two "John Smith" nodes
that share a colleague, an employer, and a project are the same person; two that share nothing are
not. Graph structure is your best disambiguator — use it.

**Always make merges reversible.** Keep the pre-merge entity ids and the edges' original endpoints so
an unmerge is possible. You will need this.

---

## 5.7 Invalidation: the algorithm

When a new fact arrives, which existing facts does it contradict?

```python
def invalidate_conflicts(new_fact: Fact, kg, now):
    # 1. Candidate conflicts: same subject + same predicate, currently valid
    candidates = kg.query_facts(
        subject=new_fact.subject, predicate=new_fact.predicate,
        valid_at=new_fact.valid_from, tenant=new_fact.tenant)

    for old in candidates:
        if old.object == new_fact.object:
            kg.reinforce(old, new_fact)          # same value restated: bump confidence
            continue

        card = CARDINALITY.get(new_fact.predicate, "many")
        if card == "one":
            # single-valued predicate: the new fact supersedes the old
            kg.invalidate(old, valid_to=new_fact.valid_from, reason=f"superseded by {new_fact.id}")
        else:
            # multi-valued: both can hold (e.g. 'speaks_language')
            if llm_contradicts(old, new_fact):
                kg.invalidate(old, valid_to=new_fact.valid_from, reason="llm_contradiction")
```

**The cardinality table is doing the heavy lifting and it should be declared, not inferred:**

```python
CARDINALITY = {
    "lives_in":  "one",     # one primary residence
    "works_at":  "one",     # simplification; "many" if you support multiple jobs
    "born_on":   "one",
    "speaks":    "many",
    "likes":     "many",
    "allergic_to": "many",
    "manages":   "many",
}
```

Asking an LLM "do these contradict?" for every pair is slow, expensive, and inconsistent. Declaring
cardinality per predicate resolves 90% of cases deterministically and leaves the LLM for the genuinely
ambiguous remainder. This is a general principle worth internalising: **push as much of the decision
into schema as you can, and reserve the model for the residue.**

---

## 5.8 Cost and latency: the honest accounting

Graph memory is expensive on the write path. Per episode you may run: entity extraction, entity
resolution (with embedding calls), fact extraction, contradiction detection, and edge invalidation.
That is several LLM calls per turn.

Mitigations that actually work:

- **Batch at session end**, not per turn (see 4.8).
- **Use a small model for extraction, a large one only for adjudication.** Extraction is a structured
  task that small models do well when the schema is tight.
- **Cache entity resolution per tenant.** The same entities recur constantly; an in-memory alias map
  per active tenant eliminates most lookups.
- **Skip the graph for low-value tenants/turns.** Two-tier: flat facts for everyone, graph promotion
  for tenants whose query patterns are multi-hop.
- **Declare cardinality** (above) to eliminate LLM contradiction checks.

Read path is usually fine: traversal on a per-tenant subgraph is small. The scaling characteristic of
agent memory is **millions of small, mostly-cold graphs**, not one giant graph — which is quite
different from classic graph database workloads and affects how you shard (chapter 07).

---

## 5.9 A worked example

```
t1 = 2025-01-10  "I just joined Acme as a backend engineer"
t2 = 2025-11-02  "Got promoted to tech lead!"
t3 = 2026-03-15  "I'm starting at Beta Corp next month"
t4 = 2026-04-20  "First week at Beta went well"
```

Resulting fact table:

| subject | predicate | object | valid_from | valid_to | recorded_at |
|---|---|---|---|---|---|
| user | works_at | Acme | 2025-01-10 | 2026-04-15 | 2025-01-10 |
| user | has_role | backend engineer | 2025-01-10 | 2025-11-02 | 2025-01-10 |
| user | has_role | tech lead | 2025-11-02 | 2026-04-15 | 2025-11-02 |
| user | works_at | Beta Corp | 2026-04-15 | NULL | 2026-03-15 |

Now the queries all work:

- *"Where do they work?"* → `valid_to IS NULL` → Beta Corp. ✓
- *"What was their role at Acme?"* → filter by Acme's validity window → tech lead (most recent within
  that window), with backend engineer as prior. ✓
- *"On 2025-06-01, what did we know?"* → system-time query → Acme, backend engineer. ✓
- *"How long were they at Acme?"* → interval arithmetic on the row. ✓ Flat memory cannot answer this
  at all.

Note row 4: `recorded_at` (March 15) precedes `valid_from` (April 15) because the user announced a
future change. On March 20, "where do they work" still correctly returns Acme.

---

## 5.10 Exercises

1. Implement the bi-temporal schema and load the worked example. Write all four queries above and
   verify each. Then add `t5 = 2026-05-01: "Actually I never worked at Acme, that was my brother"` and
   implement retraction — note it is a *different* operation from the supersession at t3.
2. Build entity resolution with the five features listed. Construct an adversarial test set with two
   people sharing a name and measure your false-merge rate. Aim for zero false merges even at the cost
   of duplicates.
3. Implement 2-hop traversal in Postgres with a recursive CTE, then in Neo4j. Benchmark both at 10k,
   100k, and 1M facts per tenant. Find where Postgres stops being adequate — for many workloads it
   never does within one tenant, which is the point.
4. Read the Zep paper (arXiv 2501.13956) end to end and write a one-page critique: what would you
   change for a workload of 10M users with ~200 facts each?

Next: `06-procedural-and-reflective.md`.
