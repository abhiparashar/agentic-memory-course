# Agentic Memory: From First Principles to Production

A self-contained course on how memory for LLM agents actually works, and how it is built at scale.

Written the way I would onboard a new L5/L6 onto a memory infrastructure team: concepts first, then
the data structures, then the systems engineering, then the failure modes that show up only at
scale, then the evaluation discipline that keeps you honest.

---

## Who this is for

You can write production Python/Go, you have shipped a service behind a load balancer, and you have
called an LLM API. You do **not** need an ML background — almost nothing in agentic memory is model
training. It is 80% distributed systems, 15% information retrieval, 5% prompt design.

That ratio is the single most important thing to internalise before you start. Teams that treat
memory as a prompting problem ship demos. Teams that treat it as a storage + retrieval + consistency
problem ship products.

---

## The one-paragraph version

An LLM is a pure function: `tokens_in -> tokens_out`. It has no state between calls. "Memory" is
therefore never inside the model — it is a system you build around the model that decides, on every
single turn, which tokens go into the context window. Everything else in this course is detail about
how you choose those tokens: where you store candidates, how you rank them, how you keep them true
over time, how you delete them when a user asks, and how you prove the whole thing works.

---

## File map

| File | What it covers | Read time |
|---|---|---|
| `01-foundations.md` | Why statelessness hurts, the memory taxonomy, cognitive-science analogies and where they break | 45 min |
| `02-context-engineering.md` | The context window as a budget. Buffers, summarisation, compaction, context rot, sub-agent isolation | 60 min |
| `03-retrieval-fundamentals.md` | Embeddings, chunking, ANN indexes, hybrid search, rerankers, RRF, filtering. The IR core | 90 min |
| `04-memory-write-path.md` | Extraction, salience, consolidation, conflict resolution, decay/forgetting. The hard half | 90 min |
| `05-temporal-and-graph-memory.md` | Bi-temporal facts, entity resolution, knowledge-graph memory, Graphiti/Zep architecture | 75 min |
| `06-procedural-and-reflective.md` | Procedural memory, reflection loops, experience replay, sleep-time compute, self-editing memory | 60 min |
| `07-systems-design.md` | Storage topology, multi-tenancy, sharding, latency budgets, caching, cost models, backfills | 90 min |
| `08-evaluation.md` | LoCoMo / LongMemEval / BEAM, offline harnesses, online metrics, regression gates, A/B design | 75 min |
| `09-security-privacy-governance.md` | Memory poisoning, indirect injection, PII, GDPR erasure across derived state, audit trails | 60 min |
| `10-case-studies.md` | MemGPT/Letta, Mem0, Zep/Graphiti, LangGraph Store/LangMem, Cognee, assistant + coding-agent memory | 75 min |
| `11-projects-small.md` | Six small projects, each 2–8 hours, with acceptance criteria | — |
| `12-projects-capstone.md` | Three capstones, each 2–6 weeks, spec'd like real design docs | — |
| `13-reading-list.md` | Books, papers, repos, courses — ordered, annotated | — |
| `14-design-review-playbook.md` | The questions I ask in a memory design review, plus a rubric | 30 min |

---

## Suggested 12-week plan

Assumes ~8 focused hours/week. Adjust freely, but do **not** skip Week 5–6; the write path is where
most engineers have the weakest intuitions.

**Weeks 1–2 — Foundations and context**
- Read `01`, `02`.
- Build Project S1 (naive buffer + summariser) and S2 (token budget allocator).
- Deliverable: a chat loop that survives 500 turns without exceeding a fixed token budget.

**Weeks 3–4 — Retrieval**
- Read `03`.
- Build Project S3 (hybrid search over conversation history, no framework).
- Deliverable: BM25 + dense + RRF + cross-encoder rerank, with recall@k measured on your own labels.

**Weeks 5–6 — The write path**
- Read `04`.
- Build Project S4 (fact extraction with ADD/UPDATE/DELETE/NOOP decisions).
- Deliverable: a memory store that correctly handles "actually, I moved to Bangalore last month".

**Weeks 7–8 — Temporal and graph memory**
- Read `05`, `06`.
- Build Project S5 (bi-temporal fact store) and S6 (reflection / sleep-time job).
- Deliverable: point-in-time queries — "what did the system believe about the user on March 1?"

**Weeks 9–10 — Production systems**
- Read `07`, `09`.
- Start Capstone C1.
- Deliverable: a memory service with p99 budgets, tenant isolation, and a working hard-delete path.

**Weeks 11–12 — Evaluation and defence**
- Read `08`, `14`.
- Finish C1, run it against LoCoMo or a self-built eval set. Write the design doc.
- Deliverable: a design doc that would survive a real design review, plus a regression suite in CI.

Then pick C2 or C3 depending on whether you want to go deep on coding agents or multi-agent systems.

---

## Ground rules for this course

**1. Build the naive version first, always.**
Every abstraction in Mem0, Zep, or Letta exists because something simpler broke. If you adopt the
abstraction before you have felt the break, you will misconfigure it. Each chapter therefore starts
with the naive implementation and shows exactly where it fails.

**2. No framework until chapter 10.**
You will write your own chunker, your own retriever, your own extractor. They will be worse than the
open-source ones. That is the point — you will know *why* they are worse.

**3. Measure everything in three units.**
Accuracy (did it recall the right thing), tokens (what did it cost per turn), latency (p50/p99).
Any memory claim quoted without all three is marketing. A system at 95% recall and 40k tokens/query
is usually worse in production than one at 88% recall and 5k tokens/query.

**4. Assume the memory is wrong.**
Design every read path so that a stale, poisoned, or hallucinated memory degrades the answer rather
than corrupting an action. This single principle will save you more incidents than any retrieval
improvement.

---

## Environment setup

Minimal, deliberately boring stack. Everything here runs on a laptop.

```bash
python -m venv .venv && source .venv/bin/activate
pip install \
  openai anthropic \
  sentence-transformers \
  rank-bm25 \
  numpy scipy scikit-learn \
  sqlalchemy psycopg[binary] \
  pgvector \
  duckdb \
  pytest \
  fastapi uvicorn
```

For the storage chapters you will want Docker:

```bash
# Postgres with pgvector — your default vector store for the whole course
docker run -d --name pgv -p 5432:5432 \
  -e POSTGRES_PASSWORD=pw pgvector/pgvector:pg16

# Neo4j — only needed for chapter 05
docker run -d --name neo4j -p 7474:7474 -p 7687:7687 \
  -e NEO4J_AUTH=neo4j/password neo4j:5
```

Deliberate choice: **start with Postgres + pgvector, not a dedicated vector DB.** At the scale you
will build in this course (< 10M vectors), Postgres is faster to operate, gives you transactions
across the memory table and the vector index, and forces you to understand the index parameters
instead of hiding them. Chapter 07 covers when to graduate.

---

## A note on the field's velocity

This space moves fast and the vendor landscape churns — licences change, community editions get
retired, and benchmark leaderboards shift every few months. Treat every specific number, star count,
and pricing tier in `10-case-studies.md` as a snapshot, and re-verify before you make a build/buy
decision. The **architectural patterns** in chapters 01–09 have been stable since 2023 and will
outlive the current crop of products; that is where to invest your reading time.

Start with `01-foundations.md`.
