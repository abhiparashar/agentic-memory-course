# 05 — Temporal and Graph Memory

> Goal: represent a world that changes. Answer "what is true now", "what was true then", "what did
> we believe then" — plus multi-hop questions that flat retrieval cannot reach.
>
> Chapter 04 ended with `valid_from` and `created_at` as two different columns. This chapter is about
> why they are two different **clocks**, and what you can answer once you accept that.

---

## 5.0 In plain words

### Two clocks, one everyday example

On **1 June** your friend tells you: *"I moved to Mumbai back in January."*

Two dates are now in play and they are both correct:

- **When it became true in the world:** 15 January.
- **When you found out:** 1 June.

A memory system with one timestamp has to pick one and throws the other away. Pick "when I found
out" and you believe your friend was in Pune until June — wrong about the world. Pick "when it became
true" and you can no longer explain why you posted them a birthday card to Pune in March — you look
like you were careless, when in fact you were acting correctly on what you knew.

Keep **both** and you can answer all three questions that matter:

| Question | Which clock | Real use |
|---|---|---|
| Where do they live *now*? | world time, latest | every normal request |
| Where did they live *in March*? | world time, as of March | "what did I do last quarter?" |
| What did I *believe* in March? | system time, as of March | "why did the agent say that?" — incident response |

That is the whole idea. "Bi-temporal" is a scary word for keeping two dates instead of one.

### The naive version and its three dead ends

```sql
-- what almost everyone ships first
CREATE TABLE facts (user_id UUID, text TEXT, updated_at TIMESTAMPTZ);
```

```
"I live in Pune"     updated_at = 2024-05-01
"I moved to Mumbai"  updated_at = 2026-06-01   ← overwrites, or sits alongside as a contradiction
```

Dead end 1 — **"where did I live last year?"** The old row is gone or indistinguishable.
Dead end 2 — **"I told you in January!"** You cannot show when you learned things, so you cannot
defend or debug a past answer.
Dead end 3 — **"I'm starting at Beta Corp next month."** With one clock this is either true now
(wrong) or ignored (also wrong). With world time it is simply a row whose `valid_from` is in the
future, and it starts applying on its own.

Every one of those is a *data model* problem. No amount of reranking fixes it.

### And the graph part, in plain words

The second half of this chapter answers a different question: *is a knowledge graph worth it?*

A flat store answers "what do I know about **X**". A graph answers "what do I know about things
**connected to** X" — "who on my team works on the service my manager mentioned?" — which needs two
hops and cannot be expressed as similarity.

The honest answer, up front: **most agent memory does not need a graph database.** It needs
*graph-shaped data* (subject–predicate–object) which fits comfortably in Postgres. Section 5.4 is the
decision rule, and it is written to talk you out of Neo4j unless you actually need it.

**The one-sentence takeaway:** store facts as triples with two clocks, keep the raw episodes they
were derived from, invalidate instead of deleting, and only add a graph engine when you can name the
traversal query that recursive SQL cannot serve.

---

## 5.1 Why flat memory fails on time

Take a user whose history contains:

```
2025-01-10  "I work at Acme as a backend engineer"
2025-11-02  "Got promoted to tech lead"
2026-03-15  "Starting at Beta Corp next month"
```

Ask: *"Where does the user work?"*

Flat vector retrieval returns all three — all similar to the query — the model sees three
contradictory statements, and it will either pick one arbitrarily, hedge, or hallucinate a merge
("tech lead at Beta Corp since 2025"). Adding a recency boost patches this until you ask *"what was
their role at Acme?"*, where the newest fact is the wrong one.

The fix is not better retrieval. Facts need time attached.

---

## 5.2 Bi-temporal modelling

Two independent time axes per fact. This is old, settled database technology — SQL:2011 standardised
system-versioned and application-time-period tables — and it is exactly the right tool here.

| Axis | Columns | Meaning | Answers |
|---|---|---|---|
| **World time** (valid time) | `valid_from` / `valid_to` | when the fact was true in reality | "where did they live in 2024?" |
| **System time** (record time) | `recorded_at` / `expired_at` | when *we* believed it | "why did the agent say that on 3 March?" |

Four timestamps per fact. That is the price of admission, and it is cheap.

Graphiti — the open-source engine behind Zep — is built on exactly this model: facts are edges with
validity windows, and "when information changes, old facts are invalidated — not deleted", so you
can "query what's true now, or what was true at any point in time"
([graphiti README](https://github.com/getzep/graphiti), [arXiv:2501.13956](https://arxiv.org/abs/2501.13956)).
Its own comparison table describes the difference from GraphRAG as "explicit bi-temporal tracking
with automatic fact invalidation" versus "basic timestamp tracking". If you build nothing else from
this chapter, build that.

### Why you need both axes, concretely

- **World time alone** cannot answer "the agent recommended a steakhouse on 3 March — did it know
  about the vegetarian preference then?" You need to know when you *learned* it.
- **System time alone** cannot handle retroactive information — the 1 June / 15 January case from
  5.0. Both dates are true and both matter.

Retroactive statements are common ("I actually moved in January", "that meeting was last Tuesday,
not today"), and they are the clearest argument for bi-temporality. Systems keyed only on
`created_at` get every one of them wrong.

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

-- the hot-path index: currently-believed, currently-true facts only.
-- Keep it small and it stays resident in cache; this is the query that runs on every turn.
CREATE INDEX ON facts (tenant_id, subject_id)
  WHERE valid_to IS NULL AND expired_at IS NULL;
```

Two invariants worth enforcing at the database level, not in application code:

```sql
-- 1. windows must make sense
ALTER TABLE facts ADD CONSTRAINT sane_window
  CHECK (valid_to IS NULL OR valid_to > valid_from);

-- 2. single-valued predicates have at most one current row (the ch04 invariant, now temporal)
CREATE UNIQUE INDEX one_current_per_predicate ON facts (tenant_id, subject_id, predicate)
  WHERE valid_to IS NULL AND expired_at IS NULL AND predicate IN ('lives_in','born_on','primary_language');
```

The second one has caught more real bugs than any prompt I have written. A reconciler that forgets to
close the old window fails loudly at insert time instead of quietly leaving the agent believing two
cities.

### The four canonical queries

These are verified against a real table — the data is the worked example from 5.9, loaded into
SQLite, and the outputs below are actual results, not illustrations.

```sql
-- 1. WHAT IS TRUE NOW
SELECT predicate, object FROM facts
WHERE subject = 'user'
  AND valid_from <= now() AND (valid_to IS NULL OR valid_to > now())
  AND expired_at IS NULL;
```
```
lives_in  Mumbai
works_at  Beta Corp
```

```sql
-- 2. WORLD-TIME TRAVEL — what was actually true on 2026-03-20,
--    according to everything we know today
SELECT predicate, object FROM facts
WHERE subject = 'user'
  AND valid_from <= '2026-03-20' AND (valid_to IS NULL OR valid_to > '2026-03-20')
  AND expired_at IS NULL;
```
```
has_role  tech lead
lives_in  Mumbai          ← we only learned this in June
works_at  Acme
```

```sql
-- 3. SYSTEM-TIME TRAVEL — what did we BELIEVE on 2026-03-20, about 2026-03-20?
--    Both axes pinned. This is the "bitemporal rectangle".
SELECT predicate, object FROM facts
WHERE subject = 'user'
  AND recorded_at <= '2026-03-20' AND (expired_at IS NULL OR expired_at > '2026-03-20')
  AND valid_from  <= '2026-03-20' AND (valid_to    IS NULL OR valid_to    > '2026-03-20');
```
```
has_role  tech lead
lives_in  Pune            ← the belief the agent actually acted on
works_at  Acme
```

**Stop and look at queries 2 and 3.** Same date, different answer: `Mumbai` versus `Pune`. Query 2
is *truth as we now understand it*; query 3 is *the belief the agent held at the time*. When a user
says "why did you send that to Pune?", query 3 is your answer and query 2 is the reason you were
wrong. A single-clock schema cannot distinguish them, which means it cannot ever explain itself.

```sql
-- 4. AUDIT TRAIL — every version of a belief, in the order we held it
SELECT id, object, valid_from, valid_to, recorded_at, expired_at
FROM facts WHERE subject='user' AND predicate='lives_in'
ORDER BY recorded_at, id;
```
```
5  Pune    2024-05-01  NULL        2024-05-01  2026-06-01   ← believed for two years, then expired
6  Pune    2024-05-01  2026-01-15  2026-06-01  NULL         ← re-asserted WITH an end date
7  Mumbai  2026-01-15  NULL        2026-06-01  NULL         ← and the new fact, backdated to January
```

Row 6 is the part people miss. Learning "I moved in January" does not just add Mumbai — it
**rewrites the shape of the Pune fact** (open-ended → closed at 15 January) while preserving the
original open-ended version for audit. Three rows, one sentence of user input, and now every
question in the table at the top of 5.0 is answerable.

I have never regretted building query 3. I have regretted not having it.

---

## 5.3 Extracting temporal information

Users state time in three ways and you must handle all three:

- **Absolute** — "on March 4th 2025" → parse directly.
- **Relative** — "two weeks ago", "last month", "since I graduated" → resolve against the message
  timestamp (`t_ref`).
- **Implicit** — "I work at Acme" → `valid_from = t_ref`, `valid_to = NULL`. Present tense implies
  current validity.

Always pass the message timestamp into the extraction prompt; Graphiti does the same, resolving
relative expressions against the episode's reference time.

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
- Relative expressions resolve against t_ref, never against the current wall clock.
"""
```

Four details that bite:

**The future case.** "I'm starting at Beta Corp next month" must not immediately override the current
employer. Store it with a future `valid_from` and the "what is true now" query excludes it until the
date arrives — then includes it automatically. **Correctness over time falls out of the query, with
no cron job flipping flags.** That is the nicest property of this model, and it is why you should
resist the urge to store a `is_current` boolean.

**`precision`.** "In 2024" (year) should not override "on 2024-03-15" (day) for the same attribute.
Carry precision and use it in the conflict policy from 4.5.

**`t_ref`, never `now()`.** Extraction runs asynchronously (4.8), sometimes minutes or hours after
the message, and backfills run *months* later. Resolving "yesterday" against the worker's clock
silently corrupts every relative date in a replay. Pass the episode timestamp explicitly and make it
a required parameter so the mistake is impossible.

**Timezones.** Store `TIMESTAMPTZ`, extract the user's zone once as an `identity` fact, and resolve
"this morning" in *their* zone. A user in IST talking to a UTC server loses or gains a day
constantly, and the symptom — "it keeps getting my dates off by one" — sounds like a model problem
and is not.

---

## 5.4 When to use a graph

Be honest about this, because a graph database carries real operational cost: another datastore to
shard, back up, monitor, upgrade, and explain to on-call.

**A graph earns its place when:**

- Your queries are **multi-hop**: "who at my company works on the project my manager mentioned?"
- **Relationships are themselves data** you query, not just attributes of one subject.
- You need **traversal retrieval** — start at an entity and walk outward — because similarity cannot
  express "connected to".
- You have **many entity types** with rich interconnection (enterprise, CRM, org knowledge).

**A graph is overkill when:**

- Your memory is 95% "facts about one user" — that is a table with a `subject_id` column.
- Your queries are single-hop attribute lookups.
- You have < 100k facts per tenant.

The middle path I recommend and have never regretted: **model your data as a graph
(subject–predicate–object triples) and store it in Postgres.** You get the semantics without
operating a second engine. Recursive CTEs handle 2–3 hops on per-tenant data perfectly well:

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

Note `AND NOT (f.id = ANY(w.path))`: cycle protection. Memory graphs have cycles (A manages B,
B collaborates with A) and without that clause the CTE runs until it exhausts memory. Depth limits
alone are not enough — you need both.

**The decision rule, stated as a test you can actually run:** write your three hardest real queries
as recursive CTEs and benchmark them at your p99 tenant size. Adopt a graph engine only when one of
them misses the latency budget from chapter 07. "It feels graph-shaped" is not a reason; a failing
benchmark is.

One more consideration that cuts against graph engines for this workload: agent memory is
**millions of small, mostly-cold graphs**, not one large hot graph. Classic graph databases are
tuned for the latter. Zep responded to precisely this by building a proprietary engine "for millions
of context graphs with low-latency retrieval" rather than using a third-party graph database
([graphiti README](https://github.com/getzep/graphiti)) — which tells you the shape of the problem
you would be taking on.

---

## 5.5 The Graphiti/Zep architecture, explained

Worth studying in detail because it is the most fully articulated production design in the open
literature: the paper is short ([arXiv:2501.13956](https://arxiv.org/abs/2501.13956)) and the code is
open source.

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

Four properties worth stealing wholesale, whatever you build:

1. **Non-lossy episodes.** Facts are *derived*, never the only copy — "every entity and relationship
   traces back to the episodes (raw data) that produced it. Full lineage from derived fact to source."
   This single property is what makes every future model upgrade tractable: you re-derive instead of
   migrating guesses.
2. **Bidirectional indices** between episodes and facts. Facts → episodes gives citation and audit;
   episodes → facts gives fast incremental updates.
3. **Bi-temporal edges with invalidation, not deletion** (5.2).
4. **Incremental construction.** New data integrates immediately "without requiring complete graph
   recomputation". Batch-rebuild designs (classic GraphRAG) are a poor fit for memory, where writes
   arrive continuously and freshness is the product.

### Community summaries: semantic memory from episodic memory

The third tier is the abstraction move from chapter 01 made concrete: raw recall at the bottom,
generated summaries at the top, and retrieval can enter at either level. Ask "what is this user
like?" and you want the community summary; ask "what did they say about the Q3 migration?" and you
want the episode. Build the bottom two tiers first — the community tier is the one to defer, because
it is the most expensive to keep fresh and the easiest to get wrong.

### Retrieval in this architecture

Three search methods, fused and reranked — the same funnel as chapter 03, with traversal added:

```
query
  ├─► cosine similarity over edge/node embeddings
  ├─► BM25 full-text over fact text
  └─► breadth-first graph traversal from seed nodes
          │
          ▼  fuse (RRF / MMR / optional cross-encoder)
          ▼  context constructor: render facts WITH their validity ranges
```

Note the last step, which is free and which almost nobody does. Render:

```
works_at(User, Acme)      [2025-01-10 → 2026-04-15]
works_at(User, Beta Corp) [2026-04-15 → present]
```

instead of two bare sentences "User works at Acme" / "User works at Beta Corp". The model can now
*reason* about currency instead of guessing which line is fresher. It is a formatting decision with
outsized quality impact, and you can adopt it today in a flat store with no graph at all.

---

## 5.6 Entity resolution in a graph, done carefully

The graph amplifies both the value and the risk of entity resolution. A correct merge unlocks
multi-hop reasoning; an incorrect merge silently fuses two people's lives — and in a graph, the
damage spreads along the edges.

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
not. **Graph structure is your best disambiguator** — which is, incidentally, the strongest argument
for graph-shaped storage even when you never traverse at read time.

**Always make merges reversible.** The unmerge is not optional; you will need it:

```sql
CREATE TABLE entity_merges (
  id UUID PRIMARY KEY, tenant_id UUID, winner_id UUID, loser_id UUID,
  score REAL, features JSONB, decided_by TEXT,     -- 'auto' | 'llm' | 'human'
  merged_at TIMESTAMPTZ DEFAULT now(), undone_at TIMESTAMPTZ
);
-- and never rewrite an edge's endpoint in place: keep original_subject_id / original_object_id
-- on the edge so an unmerge is a mechanical replay, not forensics.
```

Note `decided_by`. When you find a bad merge you want to know whether the *threshold* is wrong
(auto), the *prompt* is wrong (llm), or the *guidelines* are wrong (human). Same symptom, three
completely different fixes.

---

## 5.7 Invalidation: the algorithm

When a new fact arrives, which existing facts does it contradict?

```python
def invalidate_conflicts(new_fact: Fact, kg, now):
    # 1. Candidates: same subject + same predicate, valid at the new fact's start
    candidates = kg.query_facts(
        subject=new_fact.subject, predicate=new_fact.predicate,
        valid_at=new_fact.valid_from, tenant=new_fact.tenant)

    for old in candidates:
        if old.object == new_fact.object:
            kg.reinforce(old, new_fact)          # same value restated: bump confidence (4.6)
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

**The cardinality table is doing the heavy lifting, and it should be declared, not inferred:**

```python
CARDINALITY = {
    "lives_in":    "one",     # one primary residence
    "works_at":    "one",     # simplification; "many" if you support multiple jobs
    "born_on":     "one",
    "speaks":      "many",
    "likes":       "many",
    "allergic_to": "many",
    "manages":     "many",
}
```

Asking an LLM "do these contradict?" for every pair is slow, expensive, and inconsistent between
calls. Declaring cardinality per predicate resolves the large majority of cases deterministically and
leaves the model for the genuinely ambiguous remainder. The general principle is worth internalising
beyond memory systems: **push as much of the decision into the schema as you can, and reserve the
model for the residue.**

### Retroactive inserts: gaps and overlaps

The hard case is not "new fact now". It is **a fact arriving with a `valid_from` in the past**, which
lands in the middle of an existing timeline. Three shapes, and you must decide each deliberately:

```
existing:   [-------- Pune: 2024-05-01 → NULL --------------------->
incoming:            Mumbai from 2026-01-15

OVERLAP   → close Pune at 2026-01-15. (the 5.2 case; correct for cardinality "one")

existing:   [--- Pune: 2024-05-01 → 2025-12-01 ---]        [--- Delhi: 2026-03-01 → NULL --->
incoming:                     Mumbai: 2026-01-15 → NULL

GAP       → the incoming fact's window collides with Delhi's start. Close Mumbai at 2026-03-01
             instead of leaving two open-ended rows. Never let two "one"-cardinality rows overlap.

existing:   [--- Pune: 2024-05-01 → NULL --->
incoming:        Pune: 2023-01-01 → NULL         (earlier start, same value)

EXTEND    → do not insert. Widen the existing window's valid_from and record the new episode.
```

Implement these as an explicit interval-repair function with a test per shape. If you skip it, the
symptom is subtle and awful: `SELECT ... WHERE valid_to IS NULL` starts returning two rows, the
prompt gets both, and the agent looks confused for reasons no log explains. That is exactly what the
`one_current_per_predicate` unique index in 5.2 turns into a loud, immediate failure.

---

## 5.8 Cost and latency: the honest accounting

Graph memory is expensive on the **write** path. Per episode you may run entity extraction, entity
resolution (with embedding calls), fact extraction, contradiction detection, and edge invalidation —
several LLM calls per turn if you are careless.

Mitigations that actually work, roughly in order of payoff:

- **Batch at session end**, not per turn (4.8, 4.10).
- **Declare cardinality** (5.7) to eliminate most LLM contradiction checks. Free accuracy *and* free
  latency; do this first.
- **Small model for extraction, large model only for adjudication.** Extraction is a tight structured
  task. Note Graphiti's own warning that this pipeline "works best with LLM services that support
  Structured Output… Using other services may result in incorrect output schemas and ingestion
  failures. This is particularly problematic when using smaller models" — so pair the small model
  with real schema-constrained decoding, not hope.
- **Cache entity resolution per tenant.** The same dozen entities recur constantly; an in-memory
  alias map per active tenant removes most lookups.
- **Skip the graph for low-value tenants.** Two tiers: flat facts for everyone, graph promotion for
  tenants whose queries are actually multi-hop.

And plan for rate limits, because this is the stage that hits them: Graphiti ships with
`SEMAPHORE_LIMIT` defaulting to **10** concurrent operations specifically to avoid provider `429`s
([graphiti README](https://github.com/getzep/graphiti)). Your ingestion throughput ceiling is
usually your LLM quota, not your database — size the queue and the alerting accordingly (4.8).

The **read** path is usually fine: traversal over one tenant's subgraph is small, and Zep reports
sub-200ms retrieval at scale for its managed engine. The scaling characteristic to design for is
millions of small graphs, which is a sharding question (chapter 07), not a traversal question.

---

## 5.9 A worked example

```
t1 = 2025-01-10  "I just joined Acme as a backend engineer"
t2 = 2025-11-02  "Got promoted to tech lead!"
t3 = 2026-03-15  "I'm starting at Beta Corp next month"
t4 = 2026-06-01  "By the way I moved to Mumbai back in January"
```

Resulting fact table (this is the exact data behind the verified queries in 5.2):

| id | subject | predicate | object | valid_from | valid_to | recorded_at | expired_at |
|---|---|---|---|---|---|---|---|
| 1 | user | works_at | Acme | 2025-01-10 | 2026-04-15 | 2025-01-10 | — |
| 2 | user | has_role | backend engineer | 2025-01-10 | 2025-11-02 | 2025-01-10 | — |
| 3 | user | has_role | tech lead | 2025-11-02 | 2026-04-15 | 2025-11-02 | — |
| 4 | user | works_at | Beta Corp | 2026-04-15 | — | 2026-03-15 | — |
| 5 | user | lives_in | Pune | 2024-05-01 | — | 2024-05-01 | 2026-06-01 |
| 6 | user | lives_in | Pune | 2024-05-01 | 2026-01-15 | 2026-06-01 | — |
| 7 | user | lives_in | Mumbai | 2026-01-15 | — | 2026-06-01 | — |

Every question now has a query, and I ran all of them:

| Question | Mechanism | Answer |
|---|---|---|
| "Where do they work?" | `valid_to IS NULL` | Beta Corp |
| "What was their role at Acme?" | join on overlapping windows | backend engineer → tech lead |
| "How long were they at Acme?" | interval arithmetic on row 1 | 460 days |
| "Where did they live in March 2026?" | world-time travel | Mumbai |
| "What did we think in March 2026?" | bitemporal rectangle | Pune |
| "When did we learn about the move?" | `recorded_at` on row 7 | 2026-06-01 |

Two rows to study:

- **Row 4**: `recorded_at` (15 March) *precedes* `valid_from` (15 April) because the user announced a
  future change. On 20 March, "where do they work" still correctly returns Acme — no flag, no cron.
- **Rows 5–7**: the retroactive move. One sentence expired one belief, re-asserted it with an end
  date, and added a backdated fact. Flat memory cannot represent any part of this.

The third question — tenure — is the one to notice. **Flat memory cannot answer it at all**, at any
retrieval quality, because the information does not exist in the store. That is the difference
between a retrieval problem and a modelling problem, and it is the reason this chapter exists.

---

## 5.10 Failure modes seen in production

| Symptom | Root cause | Fix |
|---|---|---|
| Agent asserts two current employers | reconciler didn't close the old window | 5.2 `one_current_per_predicate` unique index |
| "It thinks I already started the new job" | future facts stored as current | 5.3 future `valid_from`, no `is_current` flag |
| Dates off by one, intermittently | relative expressions resolved against worker clock / server TZ | 5.3 mandatory `t_ref`, `TIMESTAMPTZ`, user zone as a fact |
| Cannot explain a past wrong answer | single clock, or `expired_at` never set | 5.2 query 3, soft-expire on every change |
| Two users' facts fused | entity merge in the ambiguous band | 5.6 DEFER band, `neighbour_overlap`, reversible merges |
| Traversal query hangs / OOMs | recursive CTE without cycle protection | 5.4 `NOT (f.id = ANY(w.path))` + depth cap |
| Ingestion backlog, 429s from provider | unbounded write concurrency | 5.8 semaphore + queue lag alerts (4.8) |
| Graph "rebuild" takes hours and blocks freshness | batch-oriented construction | 5.5 incremental updates, derive facts from episodes |
| Old facts silently gone after a model upgrade | facts treated as source of truth | 5.5 non-lossy episodes; re-derive, never migrate |

---

## 5.11 Exercises

1. Implement the bi-temporal schema and load the 5.9 example. Write all four canonical queries and
   check them against the outputs printed in 5.2 — including the deliberate `Mumbai`/`Pune`
   divergence between queries 2 and 3. If your queries agree, you have collapsed the two clocks.
2. Add `t5 = 2026-07-01: "Actually I never worked at Acme, that was my brother"` and implement
   retraction. Verify it is a *different* operation from the supersession at t3: after retraction,
   query 2 must show no Acme employment, while query 3 for March 2026 must still show it. This is
   chapter 04's outdated-vs-wrong distinction expressed in SQL.
3. Implement the three interval-repair shapes from 5.7 (overlap, gap, extend) with a test each. Then
   drop the `one_current_per_predicate` index, re-run, and observe how much harder the bug is to see
   without it.
4. Build entity resolution with the five features in 5.6. Construct an adversarial set with two
   people who share a name and measure your false-merge rate. Target zero false merges, accepting
   duplicates as the cost.
5. Implement 2-hop traversal as a recursive CTE, then in Neo4j or FalkorDB. Benchmark both at 10k,
   100k, and 1M facts per tenant. Find the crossover — for many single-tenant workloads there isn't
   one, which is the point of 5.4.
6. Render a retrieved fact set both ways — bare sentences versus facts with validity ranges — and run
   both through your agent on ten temporal questions. This is the cheapest quality win in the
   chapter; measure it so you believe it.
7. Read the Zep paper end to end and write a one-page critique: what would you change for 10M users
   averaging ~200 facts each? (Hint: start from "millions of small cold graphs" in 5.4/5.8.)

Next: `06-procedural-and-reflective.md` — memory that stores *how to do things*, and the background
jobs that make memory improve while nobody is talking.
