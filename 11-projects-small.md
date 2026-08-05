# 11 — Small Projects

Six projects, each 2–8 hours. Do them in order; each builds on the last and each maps to a chapter.

By the end you will have written, from scratch, every component of a memory system. That is the
point — the frameworks in chapter 10 will then read as familiar rather than magical.

**Ground rules:**
- No memory frameworks. Postgres, an embedding model, an LLM API, and standard libraries only.
- Every project ends with a *measurement*, not a demo.
- Keep everything in one repo; later projects import earlier ones.

---

## S1 — The Goldfish (chapter 02) · ~3 hours

**Build:** a chat loop that survives 500 turns within a fixed token budget.

**Requirements**
- Token counter, and a `ContextBudget` class with named regions and hard caps.
- Three history strategies behind one interface: `SendAll`, `SlidingWindow(n)`, `WindowPlusSummary`.
- Summaries regenerate from the durable log, never from the previous summary.
- Log per-region token occupancy for every turn.

**Test**
Generate a 500-turn conversation where a critical fact ("my account number is ACC-77213") appears at
turn 4 and is asked for at turn 480.

**Measure**
| Strategy | Answered correctly? | Total input tokens | p95 turn latency |
|---|---|---|---|

**You are done when** you can explain the exact turn at which `SlidingWindow(20)` starts failing, and
show that `WindowPlusSummary` preserves the account number through at least 10 summarisation rounds.

**Stretch:** implement `compaction_guard` from 02.3 and plot identifiers lost per summarisation round.
This is context collapse, made visible on your own data.

---

## S2 — Hybrid Retriever (chapter 03) · ~6 hours

**Build:** a retrieval stack over conversation history, with real IR metrics.

**Requirements**
- Postgres + pgvector. Store messages with embeddings and a `tsvector`.
- Three retrieval arms: dense (pgvector), lexical (BM25 via `tsvector` or `rank_bm25`), recency.
- RRF fusion with `k=60` and per-arm weights.
- Optional cross-encoder rerank stage (`cross-encoder/ms-marco-MiniLM-L-6-v2` is fine).
- MMR diversification on the final selection.

**Test**
Build a labelled set: 100 queries over your own chat export (or a public conversation dataset), with
gold message ids. Use the LLM-labelling recipe from 03.9 and hand-check 10%.

**Measure**
| Config | Recall@20 | nDCG@5 | p95 latency |
|---|---|---|---|
| dense only | | | |
| BM25 only | | | |
| RRF(dense, bm25) | | | |
| RRF + rerank | | | |
| RRF + rerank + MMR | | | |

**You are done when** you can name one query in your set that only BM25 finds and one that only dense
finds, and explain why each fails for the other.

**Stretch:** sweep `ef_search` from 10 to 400 and plot recall vs. latency. Then compare against brute
force on the same data. Note the corpus size at which the index starts winning.

---

## S3 — The Fact Store (chapter 04) · ~8 hours · *the most important one*

**Build:** extraction + reconciliation with ADD / UPDATE / DELETE / NOOP.

**Requirements**
- Typed memory categories with volatility and TTL (04.2).
- Extraction returning atomic claims with `category`, `stance`, `confidence`, `evidence_span`,
  `temporal`. Validate that `evidence_span` literally occurs in the source — reject if not.
- Prefilter: secret scanner, opt-out marker ("don't remember this"), minimum length.
- Reconciliation against the top-8 related existing memories, emitting one of the four ops.
- Soft supersession (`valid_to`, `superseded_by`) distinct from retraction (`retracted_at`).
- `NOOP` bumps confidence and resets decay.

**Test**
Implement all eight scenarios from 04.10 as a pytest suite.

**Measure**
- Scenario pass rate (target: 8/8).
- On 20 real conversations: extraction precision and recall against your own hand-labels.
- Duplicate rate: fraction of memory pairs within a tenant with cosine > 0.95 (target: < 2%).

**You are done when** scenarios 4 (hypothetical not stored), 6 (opt-out honoured), and 7
(idempotency) pass. They will all three fail on your first attempt. Write down what you changed for
each — prompt fix vs. code fix — and notice which fixes were durable.

---

## S4 — Time Machine (chapter 05) · ~5 hours

**Build:** bi-temporal fact storage with point-in-time queries.

**Requirements**
- The `facts` schema from 05.2, with world time and system time.
- Temporal extraction resolving absolute, relative, and implicit expressions against `t_ref`, plus
  correct handling of *future* validity ("starting next month").
- A declared `CARDINALITY` table driving deterministic invalidation, with LLM adjudication only for
  multi-valued predicates.
- All three canonical queries: what is true now, what was true at T, what did we believe at T.

**Test**
Load the worked example from 05.9 and verify all four questions. Then add the retraction case
("I never worked at Acme, that was my brother") and verify it behaves differently from supersession.

**Measure**
- Correctness on 20 hand-written temporal cases spanning: current state, historical state, duration,
  ordering, future-dated facts, retroactive statements.
- Compare against a flat (non-temporal) baseline on the same 20. The gap is the value of this chapter.

**You are done when** you can answer "how long were they at Acme?" — a question flat memory cannot
express at all.

---

## S5 — The Night Shift (chapter 06) · ~5 hours

**Build:** the background maintenance tier.

**Requirements**
- A job runner (a queue + worker, or a simple scheduler) with per-tenant ordering.
- Four jobs: consolidation of related memories, reflection with the usefulness filter, decay scoring
  and archival tiering, and a contradiction sweep.
- Consolidated and reflected memories record `derived_from` (required for chapter 09).
- Procedural memory file maintained by the agent, with `human-pinned` entries immune to pruning.

**Test**
Generate 300 synthetic memories across a simulated year. Run the jobs.

**Measure**
- Memory count before/after consolidation; manually rate 20 consolidated memories for information
  loss (1–5).
- Reflection insight quality: rate 20 insights for actionability (1–5). Mean below 3 means your
  usefulness filter is too weak — iterate on the filter, not the prompt.
- Contradiction count before/after the sweep.
- Conversation p95 latency with inline memory management vs. background. You should win on latency
  *and* on the S3 scenario suite; that dual win is the whole argument for a background tier.

**You are done when** background processing improves both latency and quality simultaneously.

---

## S6 — Break It (chapters 08, 09) · ~6 hours

**Build:** the eval harness and the red team.

**Part A — Eval harness**
- 100 `MemoryEvalCase`s from your own data: 20 recall, 15 multi-session, 20 knowledge update, 15
  temporal, 15 abstention, 10 preference, 5 deletion.
- Queries must be in a *new session* so session memory cannot answer them.
- LLM judge with the abstention rule from 08.6, calibrated against 100 hand-graded cases (target:
  > 90% agreement).
- Report the triple: accuracy, tokens/query, p95 latency — per category.

**Part B — Red team**
- Ten injection payloads: direct statement, content in a document the agent reads, and a tool result.
  All aim to plant a durable instruction.
- Measure how many get written to memory.
- Implement provenance + trust levels (09.3, D1) and re-measure.
- Implement the deletion cascade with derived-memory regeneration and prove a deleted fact appears in
  no summary, no consolidation, and no cached context.

**Measure**
| | Before controls | After controls |
|---|---|---|
| Payloads written to memory (of 10) | | |
| Payloads that influenced a later session | | |
| Deleted-fact residue sites found | | |

**You are done when** zero payloads become instruction-stance memories and the deletion cascade test
passes at 100%.

---

## Wiring it together

After S1–S6 you have a complete memory system. Spend an hour drawing the chapter-06.7 diagram for
*your* system, naming the file and function that owns each box. Any box you cannot name is a gap, and
the chapter it belongs to tells you what will break.

Then go to `12-projects-capstone.md`.
