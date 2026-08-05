# 03 — Retrieval Fundamentals

> Goal: build a retrieval stack you understand end to end — embeddings, indexes, lexical search,
> fusion, reranking — and know which knob to turn when quality is bad.
>
> This is the chapter with the most transferable, longest-lived knowledge. Vector DB vendors churn;
> the IR fundamentals here are from the 1970s–2010s and are not going anywhere.

---

## 3.1 Why memory retrieval ≠ document RAG

You will read a lot of RAG material. Most of it assumes a static corpus of documents. Memory
retrieval differs in five ways that break naive RAG:

| | Document RAG | Memory retrieval |
|---|---|---|
| Corpus | Static, curated | Streaming, written by the system itself |
| Item size | Paragraphs (200–1000 tok) | Sentences/facts (10–50 tok) |
| Contradictions | Rare; corpus is consistent | Constant; users change jobs, minds, preferences |
| Time | Usually irrelevant | Central — "current" vs "past" is the whole question |
| Partitioning | Global index | Hard per-user partition, always |
| Query | Explicit question | Often implicit ("book me a restaurant") |

The consequences: **you cannot rely on similarity alone** (contradictory facts are similarly
similar), **your filters are as important as your embeddings** (tenant + time), and **your items are
short**, which is bad news for dense retrieval and good news for lexical retrieval.

That last point is worth dwelling on. Dense embeddings of very short texts are noisy — "User works at
Google" and "User worked at Google" embed almost identically, and that difference is exactly what you
need. This is the core argument for hybrid search in memory systems, and it is why serious systems
fuse semantic similarity with lexical matching and entity matching rather than using cosine alone.

---

## 3.2 Embeddings, briefly but properly

An embedding model maps text to a vector such that semantically similar texts are close under cosine
similarity. What you actually need to know to make decisions:

**Dimensionality.** 384 → 1536 → 3072 are common. Higher is usually slightly better and linearly
more expensive in storage and ANN search. Many modern models support Matryoshka truncation: you can
cut a 3072-dim vector to 512 dims with modest quality loss. **Use this.** Store the full vector, but
index a truncated version for the first-stage search, then rescore the top candidates with the full
vector. Big win on index memory.

**Normalisation.** Normalise to unit length, then cosine similarity is a dot product, which is
faster and lets you use inner-product indexes. Do it once at write time.

**Asymmetric models.** Some models have separate query and document prefixes (`query: ` / `passage: `,
or task instructions). Getting this wrong silently costs 10–20% recall. It is the single most common
embedding bug I have seen in code review. Write a unit test that asserts your query encoder and
document encoder use the right prefixes.

**Domain fit.** General-purpose embeddings underperform on code, on medical text, and on
identifier-heavy content. Evaluate on *your* data before committing. A 30-minute recall@10 evaluation
on 200 hand-labelled pairs will tell you more than any leaderboard.

**Versioning.** Embeddings are not portable across models or even model versions. Store
`embedding_model_id` on every row. When you change models you must re-embed everything, and you must
be able to serve both during the migration:

```sql
CREATE TABLE memory_embeddings (
  memory_id   UUID NOT NULL,
  model_id    TEXT NOT NULL,           -- 'text-embed-3-large@2024-01'
  dim         INT  NOT NULL,
  vec         vector(1536) NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (memory_id, model_id)
);
```

One row per (memory, model) makes dual-serving trivial and rollback possible. Systems that store the
vector as a column on the memory table have painful migrations. Plan for re-embedding from day one —
you will do it at least twice.

---

## 3.3 Chunking — and why memory mostly avoids it

For document RAG, chunking is a major topic. For memory, the right answer is usually: **do not
chunk; extract.** A fact is already the right granularity.

You still need chunking in two places:

1. **Episode storage** — long conversation turns, tool outputs, uploaded documents.
2. **Domain knowledge** — the non-memory corpus your agent also searches.

Practical guidance:

- Chunk on **semantic boundaries** (turn, paragraph, function, section header), not fixed character
  counts. Fixed-size chunking splits mid-sentence and is a permanent quality tax.
- **Overlap 10–20%** for prose, **0% for structured units** (a function, a message).
- **Prepend context to each chunk before embedding.** A chunk that says "It broke again" is useless
  in isolation. Prepending `[Session 2026-03-04, topic: payments migration]` measurably improves
  retrieval. This "contextual retrieval" trick is cheap and one of the highest-ROI changes available.

```python
def contextualise(chunk: str, meta: dict) -> str:
    return (f"[session={meta['session_id']} date={meta['date']} "
            f"speaker={meta['speaker']} topic={meta.get('topic','?')}]\n{chunk}")
```

Embed the contextualised form; store and display the raw form.

---

## 3.4 ANN indexes: what the parameters actually do

You do not need to implement HNSW, but you must be able to reason about its parameters, because
defaults are frequently wrong for memory workloads.

### Brute force (flat)

Exact, no build time, O(N) per query. On modern hardware with SIMD, **exact search over 100k vectors
of dim 768 is roughly single-digit milliseconds.** Per-user memory partitions are often this small.

> **This is the most important practical insight in the chapter:** if your memory is partitioned by
> user, and the median user has 5k memories, you do not need an ANN index at all. You need a filter
> and a dot product. Teams routinely deploy a distributed vector database to solve a problem that a
> `WHERE user_id = ?` plus brute force solves exactly, faster, and with zero recall loss.

### IVF (inverted file)

Cluster vectors into `nlist` lists; at query time search `nprobe` nearest lists. Fast build, decent
recall, but **degrades badly with filtering** — if you filter to one user, most probed lists contain
nothing for that user.

### HNSW (hierarchical navigable small world)

A multi-layer proximity graph. The one you will usually use.

- `M` — edges per node. 16–32 typical. Higher → better recall, more memory (memory ≈ `dim*4 + M*8`
  bytes/vector), slower build.
- `ef_construction` — candidate list size during build. 100–400. Higher → better graph, slower build.
  Build-time only; costs you nothing at query time.
- `ef_search` — candidate list at query time. **The runtime recall/latency dial.** Tune this per
  query class; it is the knob you expose.

The filtering problem again: HNSW with a post-filter can return nothing if your filter is selective.
Pre-filtering requires index support (filtered HNSW / ACORN-style traversal). **In memory systems
your filter is almost always highly selective** (one user out of millions). Check that your store
does pre-filtering properly, or partition physically so the filter is implicit.

### Quantisation

- **Scalar (int8):** 4× smaller, ~1% recall loss. Nearly free. Do it.
- **Binary:** 32× smaller, big recall loss — but excellent as a *first stage* followed by rescoring
  with full vectors. Retrieve 200 by Hamming distance, rescore to 20 by cosine.
- **Product quantisation:** best compression, more tuning, use when memory-bound at scale.

The two-stage pattern (cheap coarse retrieval → exact rescore) recurs at every level of this stack.
Learn it once, apply it everywhere.

---

## 3.5 Lexical retrieval: do not skip this

BM25 is a bag-of-words ranking function from the 1990s. It beats dense retrieval on:

- Exact identifiers: error codes, file paths, ticket numbers, SKUs, function names.
- Rare proper nouns the embedding model never saw.
- Negations and small lexical differences that dense models smooth over.

In memory systems these are common queries. "What was that error I hit in `webhook_retry.go`?" is a
BM25 query, not a vector query.

```python
from rank_bm25 import BM25Okapi
import re

def tok(s: str) -> list[str]:
    # keep identifiers intact, also emit sub-tokens
    raw = re.findall(r"[A-Za-z0-9_./-]+", s.lower())
    out = []
    for t in raw:
        out.append(t)
        out.extend(p for p in re.split(r"[_./-]", t) if p)
    return out

bm25 = BM25Okapi([tok(m.text) for m in memories])
scores = bm25.get_scores(tok(query))
```

In Postgres you get this for free with `tsvector` + GIN, in the same transaction as your vectors —
another argument for starting there.

---

## 3.6 Fusion: RRF and why you should default to it

You have two ranked lists with incomparable scores (cosine ∈ [-1,1], BM25 ∈ [0,∞)). Do **not**
normalise and add — score distributions shift with query length and corpus, and your weights will be
wrong tomorrow.

Use **Reciprocal Rank Fusion**: it uses only ranks.

```python
def rrf(rankings: list[list[str]], k: int = 60, weights: list[float] | None = None) -> list[tuple[str, float]]:
    weights = weights or [1.0] * len(rankings)
    scores: dict[str, float] = {}
    for ranking, w in zip(rankings, weights):
        for rank, doc_id in enumerate(ranking, start=1):
            scores[doc_id] = scores.get(doc_id, 0.0) + w / (k + rank)
    return sorted(scores.items(), key=lambda kv: -kv[1])
```

`k=60` is the standard constant from the original Cormack et al. paper and is a genuinely good
default; it damps the influence of the very top ranks just enough that one retriever cannot dominate.

RRF is robust, parameter-light, and works with any number of retrievers. Add a third and fourth arm
(graph traversal, recency, entity match) and it still works. Zep's retrieval, for instance, composes
cosine similarity, BM25 full-text, and graph traversal, then fuses and reranks — that shape is the
production standard.

**When to move beyond RRF:** once you have enough click/feedback data to train a learning-to-rank
model over features (scores, recency, access count, source type, entity overlap). That is a
six-months-in problem, not a day-one problem.

---

## 3.7 Reranking

First-stage retrieval optimises recall cheaply. A reranker optimises precision expensively over a
small candidate set.

**Cross-encoder** — encodes (query, candidate) jointly. Much more accurate than bi-encoders because
it sees interaction between the two. Cost is O(candidates) model calls, so it must run on ~50–100
candidates, not 10,000.

```
query ──┐
        ├──► cross-encoder ──► relevance score
cand  ──┘
```

Typical pipeline and its budget:

```
filter (tenant, time)          → 50k candidates      < 1 ms
vector top-100 + BM25 top-100  → 200 candidates      ~10 ms
RRF fuse                       → 100 candidates      < 1 ms
cross-encoder rerank           → top 8               ~30 ms  (GPU or hosted)
MMR diversify                  → final 5             < 1 ms
```

If a cross-encoder is too slow or expensive, an **LLM reranker** (ask a small model to score or pick)
is more accurate but 10–100× the latency. Reserve it for high-value, low-QPS paths.

**Diversity (MMR).** Retrieval returns near-duplicates constantly in memory systems, because the
user said the same thing five times. Maximal Marginal Relevance trades relevance against novelty:

```python
def mmr(query_vec, cand_vecs, cand_ids, k=5, lam=0.7):
    selected, remaining = [], list(range(len(cand_ids)))
    sim_q = cand_vecs @ query_vec
    while remaining and len(selected) < k:
        if not selected:
            best = max(remaining, key=lambda i: sim_q[i])
        else:
            best = max(remaining, key=lambda i:
                lam * sim_q[i] - (1 - lam) * max(cand_vecs[i] @ cand_vecs[j] for j in selected))
        selected.append(best); remaining.remove(best)
    return [cand_ids[i] for i in selected]
```

Alternatively, dedupe *at write time* — which is strictly better, and is what chapter 04 is about.

---

## 3.8 Query construction: the underrated half

Retrieval quality is bounded by the query. In a conversation the user's literal last message is
frequently a terrible query.

**Problem 1: pronouns and ellipsis.** "What about the other one?" embeds to nothing useful.
**Fix:** query rewriting — a cheap LLM call that resolves the last turn against recent context into a
standalone query.

```python
REWRITE = """Rewrite the user's latest message as a standalone search query.
Resolve all pronouns and references using the conversation. Output only the query.

Conversation:
{recent}

Latest: {latest}
Query:"""
```

**Problem 2: implicit memory needs.** "Recommend a restaurant" contains no term matching "user is
vegetarian". Similarity search will not find it.
**Fix:** two complementary techniques.
- *Slot-based retrieval:* classify the turn's intent, and fetch the memory categories that intent
  needs (`dining → [dietary, location, budget, cuisine_prefs]`). Deterministic, fast, debuggable.
- *Query expansion:* generate 2–4 hypothetical retrieval queries from the turn and union the results
  (a multi-query / HyDE-flavoured approach).

Slot-based is what I would ship first: it is cheap, predictable, and covers the head of the
distribution. Add expansion for the tail.

**Problem 3: time-scoped queries.** "What did we decide last week?" needs a time filter, not
similarity. Parse temporal expressions into filter predicates rather than hoping the embedding
captures "last week". Chapter 05 goes deep here.

---

## 3.9 Measuring retrieval (do this before optimising anything)

You cannot improve what you have not measured, and end-to-end answer quality is too noisy to guide
retrieval work. Build a **retrieval-only** eval set.

```python
# gold: query -> set of memory_ids that should be retrieved
def recall_at_k(results, gold, k):        return len(set(results[:k]) & gold) / max(len(gold), 1)
def precision_at_k(results, gold, k):     return len(set(results[:k]) & gold) / k
def mrr(results, gold):
    for i, r in enumerate(results, 1):
        if r in gold: return 1 / i
    return 0.0
def ndcg_at_k(results, gains, k):
    import math
    dcg  = sum(gains.get(r, 0) / math.log2(i + 1) for i, r in enumerate(results[:k], 1))
    ideal = sorted(gains.values(), reverse=True)[:k]
    idcg = sum(g / math.log2(i + 1) for i, g in enumerate(ideal, 1))
    return dcg / idcg if idcg else 0.0
```

How to get labels without a labelling team:
1. Take 200 real queries from logs.
2. Retrieve top-50 with a deliberately generous union of every retriever you have.
3. Have a strong LLM label each (query, candidate) as relevant/not, with a written rubric.
4. Spot-check 10% by hand — if human/LLM agreement is above ~90%, trust the rest.

That is a day of work and it converts retrieval tuning from vibes into engineering. **Do it before
you write another retriever.**

Track two numbers separately and never average them: **recall@50 of the first stage** (are the right
candidates even reaching the reranker?) and **nDCG@5 after reranking** (is the reranker ordering
them correctly?). They fail for completely different reasons and have completely different fixes.

---

## 3.10 A complete hybrid retriever (reference implementation)

```python
from dataclasses import dataclass

@dataclass
class Candidate:
    id: str
    text: str
    score: float
    source: str            # 'dense' | 'lexical' | 'graph' | 'recency'

class HybridRetriever:
    def __init__(self, store, embedder, bm25, reranker=None):
        self.store, self.embedder, self.bm25, self.reranker = store, embedder, bm25, reranker

    def retrieve(self, query: str, tenant: str, *, k=8, now=None, filters=None):
        filters = {"tenant_id": tenant, **(filters or {})}

        qv = self.embedder.encode_query(query)
        dense   = self.store.vector_search(qv, filters=filters, limit=100)
        lexical = self.bm25.search(query, filters=filters, limit=100)
        recent  = self.store.recent(filters=filters, limit=25)   # cheap recency arm

        fused = rrf(
            [[c.id for c in dense], [c.id for c in lexical], [c.id for c in recent]],
            weights=[1.0, 0.8, 0.3],
        )[:60]

        cands = self.store.get_many([cid for cid, _ in fused], tenant=tenant)

        if self.reranker:
            scored = self.reranker.score(query, [c.text for c in cands])
            cands = [c for _, c in sorted(zip(scored, cands), key=lambda t: -t[0])]

        cands = self._apply_temporal_priors(cands, now)
        return mmr_ids(qv, cands, k=k, lam=0.7)

    def _apply_temporal_priors(self, cands, now):
        """Down-weight superseded facts; mild recency boost for volatile categories."""
        out = []
        for c in cands:
            if c.valid_to is not None:            # superseded — keep only if query is historical
                c.score *= 0.2
            if c.category in VOLATILE:            # e.g. location, job, current project
                c.score *= recency_multiplier(c.valid_from, now)
            out.append(c)
        return sorted(out, key=lambda c: -c.score)
```

Note the recency arm in the fusion. In memory systems, "the most recent thing the user told me" is a
strong prior that pure similarity search will not capture, and adding it as a low-weight third
retriever is cheap and effective.

---

## 3.11 Common failure modes and their fixes

| Symptom | Likely cause | Fix |
|---|---|---|
| Recall fine, answers wrong | Contradictory facts both retrieved | Temporal validity + supersession (ch. 04/05) |
| Retrieves near-duplicates | No write-time dedup | Dedup on write; MMR as a band-aid |
| Misses exact identifiers | Dense-only retrieval | Add BM25 arm |
| Good offline, bad online | Query distribution mismatch | Build eval from real logs, not synthetic |
| Latency spikes at p99 | Unfiltered ANN + post-filter | Pre-filter or physically partition |
| Quality drops after model upgrade | Mixed embedding versions | `model_id` per vector; dual-serve during migration |
| Recall drops for new users | Cold start | Fall back to session-only + explicit onboarding prompts |
| Slowly degrading recall over months | Index not rebuilt after mass deletes | Scheduled reindex; monitor tombstone ratio |

---

## 3.12 Exercises

1. Implement brute-force cosine over 100k random vectors. Measure latency. Compare against pgvector
   HNSW with `ef_search` swept from 10 to 400, plotting recall vs. latency. Find the point where the
   index actually starts to pay for itself. It is later than you think.
2. Build the labelled eval set described in 3.9 from your own chat logs. Measure dense-only,
   BM25-only, and RRF-fused recall@20. In my experience fusion beats both by a wide margin on memory
   data; verify on yours.
3. Break the asymmetric-prefix rule on purpose and measure the recall loss. Then write the unit test
   that would have caught it.
4. Take 50 real conversational turns and measure retrieval quality with (a) the raw turn as query and
   (b) an LLM-rewritten standalone query. Quantify the gap; it is usually the biggest single win in
   this chapter.

Next: `04-memory-write-path.md` — the hard half.
